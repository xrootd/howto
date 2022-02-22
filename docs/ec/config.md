### The backend xrootd storage cluster

Since the backend xrootd cluster isn't aware of EC, there is no extra xrootd 
configuration other than setting up a plain xrootd cluster using a relatively new
xrootd release (e.g. xrootd 5.3.0+) . There is a few things to consider for HW 
configuration:

1. The xrootd cluster needs at least ** n + m ** data servers.
2. EC already provides data redundancy. This reduces the need of using RAID.

### Enabling EC in xrootd clients

Xrootd clients that directly facing the backend storage will need to load the EC 
library. The clients include adminsitrative tools `xrdcp/xrdfs/xrootdfs` and user 
facing xrootd proxy. To enable EC in xrootd clients:

1. use a xrootd release that includes `libXrdEC.so`.
2. create a file /etc/xrootd/client.plugins.d/xrdec.conf and put the following in
   this file:
```
url = root://*
lib = XrdEcDefault
enable = true
nbdta = 8
nbprt = 2
chsz = 1048576
```
Here `nbdtaa` refers to the number of data chunks in a stripe, and `nbprt` refers
to the number of parity chunks in a stripe. `chsz` referst to the chunk size (Bytes).

### Configuring a xrootd proxy using EC.

In addition to the requirement in the [Enabling EC in xrootd clients](#enable-ec-in-xrootd-clients) 
section, please refers to the 
[WLCG TPC example configuration](../../tpc/#an-example-of-wlcg-tpc-configuration-with-x509-authentication)
document for xrootd proxy (DTN) configuration.
