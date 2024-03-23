# Demo driver for Xrootd to interact with the HPSS

A prelimnary driver here shows how Xrootd interacts with both the HPSS backend and clients who 
use the Xrootd's Prepare Interface. It is intended to be an example to demonstrate the workflow 
when FTS transfer files in and out of a Tape Endpoint using the ***root*** protocol (also see the 
discussion on the WLCG Tape REST API). It is not expected to scale up for production
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

# Tech details

gfal and FTS expects a json response for `query` and `stage`. The following describe the json response.
Note the Xrootd Prepare Interface does not support HPSS `migrate`. HPSS does the migration 
from disk buffer to tape automatically.

## Json response to `stage` and `query`

***gfal-bringonline*** actually issues several commands via the Xrootd Prepare Interface:

- a "stage" command for one or more files.
- gfal-bringonline does not wait for the `stage` command to finish. It immediately issue a `query` command. 

Assume the file name is $lfn and request id is $request_id

<ol>
<li>
If the file doesn't exist in HPSS, the json response should be:

```
{ "request_id": "$request_id",
  "responses": [{"path": "$lfn", 
                 "path_exists": false, 
                 "error_text": "", 
                 "on_tape": false, 
                 "online": false},
               ]
}
```

Note the comma before ']': since gfal-bringonline may include more than one files, the `responses` list 
should include all of them.<p>
gfal-bringonline will not issue more query for a file if it gets a <b>"path_exist": false</b> response.
</li>
<li>
If a file exists in HPSS. but the staging is still running, the json response should be:

```
{ "request_id": "$request_id",
  "responses": [{"path": "$lfn", 
                 "path_exists": true, 
                 "error_text": "",
                 "on_tape": true, 
                 "requested": true, 
                 "has_reqid": true, 
                 "req_time": "$req_time"},
               ]
}
```

$req_time is the time stage request started. It doesn't need to be accurate.<p>
gfal-bringonline will wait for 2^n seconds for files that with a response like the above. n=0,1,2,3,...   
</li>
<li>
If the file exists in HPSS, and the staging has completed, the output of the <b>hsi ls -X</b> 
should be parsed to determine if the file is on disk or on tape. The json resonse should be:

```
{ "request_id": "$request_id",
  "responses": [{"path": "$lfn", 
                 "path_exists": true, 
                 "error_text": "",
                 "on_tape": true(or false), 
                 "online": true(or false},
               ]
}
```

Here "online" refers to whether the file is in disk buffer and can be accessed as a disk file.<p>
Once gfal-bringonline get a <b>"online":true</b> response, it no longer include that file in future query.
</li>
</ol>

One tricky thing is to determine whether the staging is still running. The script uses a marker file 
/tmp/stage.$request_id to record:

- The staging request_id.
- That the staging of that group of files is going in. When the staging is completed, the marker file 
will be deleted.
- The timestamp of the marker file. It is used as the time of the staging requests.

## Migration to tape

The response to query (above) will also info  ***gfal-archivepoll*** that a file has a copy on tape.
