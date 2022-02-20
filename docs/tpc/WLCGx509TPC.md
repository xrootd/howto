### The many different use cases

Xrootd is used in a wide range of cases to provide TPC services. The example configuration files below 
will try to covers several use cases. Sorting out these use cases and concepts behind them will help
understand the example configuration file.

How xrootd TPC service will access the storage?

  * If storage is not mounted to the xrootd TPC service node, the TPC node functions as a Xrootd DTN
    (a.k.a xrootd proxy, or gateway to a backend storage - on LAN or WAN).
  * If storage is mounted to the xrootd TPC node as a posix file system, the TPC node functions
    as a regular xrootd storage server. However, there are two sub-categories:
    - The storage is a local posix file system (only mounted to that particular TPC node, e.g. XFS 
      filesystem on a disk-array)
    - The storage is a shared posix file system (mount on many nodes, e.g Lustre or GPFS).

A single xrootd TPC service node, or a cluster. A cluster is how xrootd scales. In a cluster, there are

  - Several server nodes. Their roles in the cluster are either `server` or `proxy server`. The latter
    refers to the scenario where xrootd TPC service function as DTNs.
    + If server nodes mount local filesystems, then each of they have access to a different
      set of data files.
    + If server nodes mount a shared filesystem, then they all have access to the same set
      of files
    + If server nodes are `proxy server`, they usuall have access to the same set of files.
  - A redirector node. Its role is either a `manager` or `proxy manager`. A redirector sends a client to 
    a `server` or `proxy server`, based on which one has the files wanted by the client. (A redirector
    is usually lightweight, require very little resource).

An example WLCG TPC setup will cover the following:

  1. TPC services described in the above use case scenarios: 
    - `DTN` vs `local posix file system` vs `share posix file system`
    - a single TPC node or a cluster.  
  1. Use X509/VOMS for authentication and authorization
  1. Support HTTP TPC via the macaroon tokens
  1. Support Xrootd TPC via X509 proxy delegation

### Requirements
1. A copy of host certificate at /etc/grid-security/xrd/{xrdcert.pem, xrdkey.pem}. These 
files should be owned by user xrootd (or equivalent). 
1. CA certificates (default /etc/grid-security/certificates)
1. voms certificates (default /etc/grid-security/vomsdir)
1. RPMs: xrootd xrootd-voms voms libmacaroons (and their dependences). Other RPMs may also be required.
1. (optional) TCmalloc (gperftools-libs) or JEmalloc (jemalloc) may provide better memory management for xrootd.

One may also need to set nproc >= 8192 and nofile >= 8192 if the systems will be under high load.

### Example Xrootd configuration file
We expect TLS be enabled on all nodes, include the redirectors.
```
# If there is a redirector
# "redirector" should be full qualified DNS name (e.g. hostname -f). 
set redirector = <myrdr_hostname>
all.manager $(redirector):1213

xrd.port 1094
xrootd.chksum adler32

all.adminpath /var/spool/xrootd/var/spool
all.pidpath   /var/run/xrootd/var/run

all.export <my.storage.path e.g. /data/dir1>
# For DTN only
# Expect the backend xrootd door to have the same all.export as the above
#pss.origin <mybackend.storage.xrootd.door>:<port>

# TLS:
xrd.tls /etc/grid-security/xrd/xrdcert.pem /etc/grid-security/xrd/xrdkey.pem
xrd.tlsca certdir /etc/grid-security/certificates
xrootd.tls capable all

if $(redirector)
    # "manager" role: for a cluster
    #
    # Manager (rediector) role
    # For a 'proxy manager'
    #all.role proxy manager
    # For a 'manager'
    all.role manager

    # For Xrootd DTN, or Xrootd storage on a shared file system
    #cms.dfs lookup distrib redirect immed

    # For Xroodt DTN
    #pss.ckslib adler32 /usr/lib64/libXrdPss.so
else
    # "server" role: for both single node and cluster
    #
    # config the Role: server / proxy server, standalone or cluster
    # For a standalone 'proxy server'
    #ofs.osslib /usr/lib64/libXrdPss.so
    # For a standalone 'server'
    #NOTHING
    # For a 'proxy server' in a cluster
    #all.role proxy server
    # For a 'server' in a cluster
    all.role server

    # Enable security
    xrootd.seclib /usr/lib64/libXrdSec.so

    # X509 VOMS security in xroot protocol
    sec.protparm gsi -vomsfun:/usr/lib64/libXrdSecgsiVOMS.so -vomsfunparms:dbg
    sec.protocol /usr/lib64 gsi -dlgpxy:1 -exppxy:=creds -ca:1 -crl:3 -gridmap:/dev/null

    # Xrootd TPC
    # For 'proxy server': pgm /etc/xrootd/xrdcp-tpc.sh
    #ofs.tpc fcreds ?gsi =X509_USER_PROXY ttl 60 70 xfr 100 autorm pgm /etc/xrootd/xrdcp-tpc.sh
    # For 'server': 
    ofs.tpc fcreds ?gsi =X509_USER_PROXY ttl 60 70 xfr 100 autorm pgm /usr/bin/xrdcp

    # Authorizatoin
    acc.audit deny
    acc.authdb /etc/xrootd/auth_file
    acc.authrefresh 60
    ofs.authorize
fi

# Config HTTP
if ($redirector)
    http.desthttps yes
fi

# HTTP Protocol
if exec xrootd
    xrd.protocol http libXrdHttp.so
fi
http.selfhttps2http no

# X509 VOMS in HTTP(s) protocol
http.staticpreload http://static/robots.txt /etc/xrootd/robots.txt
http.secxtractor libXrdVoms.so
http.header2cgi Authorization authz

# Set all.sitename in order to use Macaroon
all.sitename MYSITE

http.exthandler xrdtpc libXrdHttpTPC.so
http.exthandler xrdmacaroons libXrdMacaroons.so
macaroons.secretkey /etc/xrootd/macaroon-secret
ofs.authlib libXrdMacaroons.so
```

### An example `xrdcp-tpc.sh` 
Please set permission of this script to 755
```
#!/bin/sh
set -- `getopt S: -S 1 $*`
while [ $# -gt 0 ]
do
  case $1 in
  -S)
      ((nstreams=$2-1))
      [ $nstreams -ge 1 ] && TCPstreamOpts="-S $nstreams"
      shift 2
      ;;
  --)
      shift
      break
      ;;
  esac
done

src=$1
dst=$2
xrdcp --server $TCPstreamOpts -f $src root://$XRDXROOTD_PROXY/${dst}
```

### The access control file `/etc/xrootd/auth_file`:
```
= atlasprod o: atlas g: /atlas r: production
x atlasprod <my.storage.path> rwildn
g /atlas <my.storage.path> rl
g /cms <my.storage.path> rl
g /dteam <my.storage.path>/dteam/doma rwild
```
1. line 1: define a special compound id "atlasprod" to identify all users within ATLAS VO with "/atlas/Role=production".
1. line 2: grant this "atasprod" access permission 'rwildn'
1. line 3: grant all other ATLAS users (all of those in VO group /atlas) permission 'rl'
1. line 4: grant all /cms users (all users in VO group /cms) permission 'rl'
1. line 5: grant all /dteam users (all users in VO group /dteam) permission 'rwild'
1. default is deny access

### The `robots.txt`
It is there to ask the internet search engines to stay away:
```
User-agent: *
Disallow: / 
```

### Generating the macaroon secret
Run command `openssl rand -base64 -out macaroon-secret 64`. All nodes, include redirectors 
should have the same macaoon secret.

### Ports
For a standalone Xrootd, xroot and https can have different ports. For a cluster, xroot and https can use different
ports only on the redirector. On data servers, xroot and https have to use the same port.

### References:
1. WLCG Xrootd 3rd Party Copy (TPC), https://twiki.cern.ch/twiki/bin/view/LCG/XrootdTpc
1. HTTP TPC, https://twiki.cern.ch/twiki/bin/view/Main/XRootDoverHTTP#Enable_Third_Party_Copy
1. Macaroons support, https://twiki.cern.ch/twiki/bin/view/Main/XRootDoverHTTP#Macaroons_Support
