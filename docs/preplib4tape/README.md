# Demo driver for Xrootd to interact with the HPSS

A prelimnary driver here shows how Xrootd interacts with both the HPSS backend and clients who 
use the Xrootd's Prepare Interface. It is intended to be an example to demonstrate the workflow 
when FTS transfer files in and out of a Tape Endpoint using the ***root*** protocol (also see the 
discussion on the WLCG Tape REST API at the end). It is not expected to scale up for production
use.

This demo driver will 

1. Use HSI v8.3 to query info related to a file in the HPSS storage hierarchy. 

    - It assumes the HPSS storage path is /DIR
    - It parses the output of HSI v8.3 (other versions of HSI may output in different format)

2. Use HPSSFS-FUSE mount to copy data in and out of HPSS. The posix mount is used to simply
the example.

    - It assumes the HPSS storage is mounted on via HPSSFS-FUSE at /hpssfs/DIR

3. Clients will use xrdfs/gfal/FTS to interact with the Xrootd endpoint

    - Use ***gfal-archivepoll*** and ***gfal-bringonline*** to trigger data migration and staging with HPSS
    - For FTS (... detail to come)

# Configuration and the driver

A few configuration lines are needed in Xrootd. They are

```
all.export /hpssfs/DIR
ofs.preplib /usr/lib64/libXrdOfsPrepGPI.so -admit all -run /etc/xrootd/prepcmd.sh
```

The [`prepcmd.sh`](prepcmd.sh.txt) is the driver written in Shell script. 

# WLCG Tape REST API

Xrootd currently does not support WLCG Tape REST API. However, as showed above, Xrootd Prepare API
provides the equivalent functionalities. A plugin to Xrootd can be developed to translate the WLCG 
Tape REST API to Xrootd Prepare commands. With that plugin, we expect it will work with the above 
driver.
