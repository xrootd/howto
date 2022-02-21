Xrootd TPC using rendezvous key works this way:

1. The client authenciates with both the data source and the destination
  (using GSI or others) and obtains authorization for read and write.
1. The client supplies a rendezvous key to both the source and destination.
1. The destination initiates a pull request using a pre-configured program
  or script.
1. In the pull request, the destination authenticates against the source.
   It will then use the rendezvous key for authorization. 
1. The destination pulls the data file.

An example config file for TPC on a single Xrootd storage node like this 
the following:
```
all.adminpath /tmp/xrootd/var/spool
all.pidpath   /tmp/xrootd/var/run

oss.localroot /tmp
all.export /xrootd

# Enable checksum
xrootd.chksum adler32

# Config TLS
xrd.tls /etc/grid-security/xrd/xrdcert.pem /etc/grid-security/xrd/xrdkey.pem
xrd.tlsca certdir /etc/grid-security/certificates refresh 8h
xrootd.tls capable all -data

#sec.level all compatible

# Enable Security
xrootd.seclib libXrdSec.so

# Enable "gsi" security
sec.protparm gsi -vomsfun:libXrdVoms.so -vomsfunparms:dbg
sec.protocol gsi -ca:1 -crl:3 -gridmap:/dev/null

# Authorizaton
acc.audit deny
acc.authdb /etc/xrootd/auth_file
acc.authrefresh 60
ofs.authorize 1

# Xrootd TPC using rendezvous key
ofs.tpc logok autorm pgm /etc/xrootd/xrdcp-tpc.sh

# Env var needed by the above TPC script.
setenv X509_USER_CERT = /etc/grid-security/xrd/xrdcert.pem 
setenv X509_USER_KEY = /etc/grid-security/xrd/xrdkey.pem 
```

We will need to enable TLS so that we can securely transport the rendezvous key
from the client to both source and destination. This also means that the xrootd
program needs to have access to the `/etc/security/xrd/{xrdcert.pem, xrdkey.pem}`
(a copy of host cert and key).

The destination will run the TPC program (xrdcp-tpc.sh). The program authenticates
againt the source (using GSI), it then uses the rendezvous key for authorization.
After that, it pulls the data. The `xrdcp-tpc.sh` looks like the following.
```
#!/bin/sh

# To work with data source's GSI, a X509 proxy is needed. It can be any X509 
# proxy - the authorization uses the rendezvous key, not the X509 proxy.
export X509_USER_PROXY=/tmp/x509up_u$(id -u)

# Note that the main xrootd config file exported the location of 
# X509_USER_CERT and X509_USER_KEY
xrdgsiproxy init \
    -cert $X509_USER_CERT \
    -key $X509_USER_KEY \
    -bits 1024 \
    -f $X509_USER_PROXY > /dev/null 2>&1

unset X509_USER_CERT
unset X509_USER_KEY
xrdcp -f $@
```

Please make sure the above script has `u+rx` bits set.

Note that for xrootd TPC, both sides (source and destination) needs to be 
configured to support xrootd TPC, whereas for HTTP TPC, only one side (usually
the desitination) needs to be configured to support HTTP TPC.

