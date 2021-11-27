Xrootd TPC using rendezvous key works this way:

1. The client authenciated with both the data source and the destination,
  and gets authorization for read and write.
1. The client supplies a rendezvous key to both the source and destination.
1. The destination initiate a pull request using a pre-configured program
  or script.
1. In the pull request, the destination authenticates against the source.
   It will then use the rendezvous key for authorization. 
1. The destination pull the data file.

An example config file for both source and destination looks like this:
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

# Enable "sss" security, the main user facing security (can be others)
sec.protocol sss -r 30 -s /etc/xrootd/mysss.keytab

# Enable "gsi" security, for destination TPC program/script to authenticate 
# with the source.  
sec.protparm gsi -vomsfun:libXrdVoms.so -vomsfunparms:dbg
sec.protocol gsi -ca:1 -crl:3 -gridmap:/dev/null

# Authorizaton
acc.audit deny
acc.authdb /etc/xrootd/auth_file
acc.authrefresh 60
ofs.authorize 1

# Xrootd TPC
ofs.tpc logok autorm pgm /etc/xrootd/xrdcp-tpc.sh

# Env var needed by the above TPC script.
setenv X509_USER_CERT = /etc/grid-security/xrd/xrdcert.pem 
setenv X509_USER_KEY = /etc/grid-security/xrd/xrdkey.pem 
```

We will need to enable TLS so that we can securely transport the rendezvous key
from the client to both source and destination. This also means that the xrootd
program needs to have access to the `/etc/security/xrd/{xrdcert.pem, xrdkey.pem}`
(a copy of host cert and key).

The main user facing security method (e.g. "sss") can be any kind of security 
method available in xrootd, including "gsi" (in which case the main and secondary
security method are merged).

The secondary security "gsi" is conveniently used here to allow the TPC program
(xrdcp-tpc.sh) to
authenticate againt the source. It will uses the rendezvous key for 
authorization. The `xrdcp-tpc.sh` looks like the following.
```
#!/bin/sh

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


