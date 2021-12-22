An example WLCG TPC setup will cover the following features:

1. Work with both Xrootd storage system (on Posix local/shared file systems), or Xrootd proxy (aka, DTN)
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
    # Manager (rediector) role
    # For a Xrootd storage system: all.role manager
    # For a Xrootd DTN: all.role proxy manager
    all.role manager

    # For Xrootd DTN, or Xrootd storage on a shared file system
    #cms.dfs lookup distrib redirect immed

    # For Xroodt DTN
    #pss.ckslib adler32 /usr/lib64/libXrdPss.so
else
    # Data server / proxy server (DTN) Role
    # For a standalone Xrootd storage, comment out
    # For a Xrootd storage cluster: all.role server
    # For a standalone Xrootd DTN: ofs.osslib /usr/lib64/libXrdPss.so
    # For a Xrootd DTN cluster: all.role proxy server
    all.role server
    #ofs.osslib /usr/lib64/libXrdPss.so
    #all.role proxy server

    # Enable security
    xrootd.seclib /usr/lib64/libXrdSec.so

    # X509 VOMS security in xroot protocol
    sec.protparm gsi -vomsfun:/usr/lib64/libXrdSecgsiVOMS.so -vomsfunparms:dbg
    sec.protocol /usr/lib64 gsi -dlgpxy:1 -exppxy:=creds -ca:1 -crl:3 -gridmap:/dev/null

    # Xrootd TPC
    # Xrootd storage: pgm /usr/bin/xrdcp  (this is the default)
    # Xrootd DTN: pgm /etc/xrootd/xrdcp-tpc.sh
    ofs.tpc fcreds ?gsi =X509_USER_PROXY ttl 60 70 xfr 100 autorm pgm /usr/bin/xrdcp
    #ofs.tpc fcreds ?gsi =X509_USER_PROXY ttl 60 70 xfr 100 autorm pgm /etc/xrootd/xrdcp-tpc.sh

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
