# Setting Up a Network IO Monitoring Dashboard in Grafana

The page provides a prototype setting to display Xrootd Network IO monitory info on a Grafana
dashboard. It assume that a functioning `Telegraf/InfluxDB/Grafana` system is available. 

## Setting in Xrootd

### Monitoring upload/write and download/read type IO traffic

Data from Xrootd summary monitoring's `Link Summary Data` will be used to monitor this type of traffic. To do so, 
add the following line to your Xrootd configuration file:
```
xrd.report 127.0.0.1:9300 every 60s link
```
The directive says to send `link` related info (bytes in and out of Xrootd to WAN) every 60s to host 127.0.0.1 
at UDP port 9300

The monitoring info it will send out is in XML format, like this:
```
<statistics tod="1725485991" ver="v5.6.9" src="sdfdtn005.sdf.slac.stanford.edu:2094" tos="1725485980" pgm="xrootd" ins="atlas" pid="482140" site="SLAC">
    <stats id="link">
        <num>1</num>
        <maxn>4</maxn>
        <tot>14</tot>
        <in>158400</in>
        <out>85782</out>
        <ctime>14</ctime>
        <tmo>0</tmo>
        <stall>0</stall>
        <sfps>0</sfps>
    </stats>
</statistics>
```

### Monitoring TPC traffic

Data to be used to monitor TPC traffic are detail monitoring's `TPC g-Stream`. To collect these info, add the 
following line to the Xrootd configuration file:
```
xrootd.mongstream tpc use flush 60s maxlen 65534 send json insthdr 127.0.0.1:9300
```

The data format is in JSON. It may include multiple JSON block separated by `\n`. The following json may occure 
at the start of the Xrootd service and occassionally at other times: 
```
{
    "code": "=",
    "pseq": 1,
    "stod": 1728672599,
    "sid": 188905941001298,
    "src": {
        "site": "SLAC",
        "host": "sdfdtn005.sdf.slac.stanford.edu",
        "port": 2094,
        "inst": "atlas-slac",
        "pgm": "xrootd",
        "ver": "v5.6.9"
    }
}
```
The following json will be sent regularly whenever there are one or more completed TPC transfers
```
{
    "code": "g",
    "pseq": 1,
    "stod": 1728672599,
    "sid": 188905941001298,
    "src": {
        "site": "SLAC",
        "host": "sdfdtn005.sdf.slac.stanford.edu",
        "port": 2094,
        "inst": "atlas-slac"
    },
    "gs": {
        "type": "P",
        "tbeg": 1728672619,
        "tend": 1728672692
    }
}
{
    "TPC": "xroot",
    "Client": "yangw.3206895:83@[2620:114:d000:55a3:42a6:b7ff:fe97:1cd0].sdf.slac.stanford.edu",
    "Xeq": {
        "Beg": "2024-10-11T18:50:20.728522Z",
        "End": "2024-10-11T18:50:57.635623Z",
        "RC": 0,
        "Strm": 1,
        "Type": "pull",
        "IPv": 4
    },
    "Src": "xroot://se0.oscer.ou.edu:9090//ourdisk/hpc/xrd_test/a",
    "Dst": "xroot://sdfdtn005.sdf.slac.stanford.edu:2094//xrootd/dteam/a",
    "Size": 10485760
}
{
    "TPC": "http",
    "Client": "yangw.3206895:83@[2620:114:d000:55a3:42a6:b7ff:fe97:1cd0].sdf.slac.stanford.edu",
    "Xeq": {
        "Beg": "2024-10-11T18:52:21.518724Z",
        "End": "2024-10-11T18:52:58.834753Z",
        "RC": 0,
        "Strm": 1,
        "Type": "pull",
        "IPv": 4
    },
    "Src": "https://se0.oscer.ou.edu:9090//ourdisk/hpc/xrd_test/b",
    "Dst": "https://sdfdtn005.sdf.slac.stanford.edu:2094//xrootd/dteam/b",
    "Size": 104757600 
}
```

Note Xrootd server will only send these data if it drives the TPC event, e.g. it is the transfer destination of
a TPC pull request, or the transfer source of a TPC push request.

## Run a collector that saves info for Telegraf

For both types of data collected, we will send the totoal bytes to Telegraf, that is, total bytes of upload/write
and download/read, and total bytes of TPC pull and push (in both xrootd and http protocols). 

The following python script running on 127.0.0.1 will receive both types of Xrootd monitoring info and and save 
the data in the InfluxDB line protocol format to a file (`/tmp/xrdmon.telegraf.log`).
```
#!/usr/bin/python3

import socket
import datetime, random
import threading
import xml.etree.ElementTree as ET
import json
from concurrent.futures import ThreadPoolExecutor

serverSocket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
serverSocket.bind(('', 9300))

logfile = "/tmp/xrdmon.telegraf.log"
nlines2keep = 720

loglock = threading.Lock()
tpclock = threading.Lock()

ibytes = {}
obytes = {}

def gstreamTPCMon(message, logfile, loglock, tpclock):
    #print("Calling gstreamTPCMon")
    messages = message.split('\n')

    jdata = []
    for i in range(0, len(messages)):
        try:  # ignore non-json lines (e.g. the last "empty" line)
            x = json.loads(messages[i])
            jdata.append(x)
        except:
            pass

    try:
        # in case it receive something like this, ignore it
        #  ['{"code":"=","pseq":3,"stod":1728535536,"sid":188905941001298,"src":{"site":"SLAC","host":"sdfdtn005.sdf.slac.stanford.edu","port":2094,"inst":"atlas-slac","pgm":"xrootd","ver":"v5.6.9"}}\x00']

        if jdata[0]['code'] != 'g':
            return
    except:
        pass

    hostport = jdata[0]['src']['host'] + ":" + str(jdata[0]['src']['port'])
    instance = jdata[0]['src']['inst']
    sitename = jdata[0]['src']['site']

    if not hostport in ibytes.keys():
        ibytes[hostport] = {}
        ibytes[hostport][instance] = 0
    elif not instance in ibytes[hostport].keys():
        ibytes[hostport][instance] = 0

    if not hostport in obytes.keys():
        obytes[hostport] = {}
        obytes[hostport][instance] = 0
    elif not instance in obytes[hostport].keys():
        obytes[hostport][instance] = 0

    for i in range(1, len(jdata)):
        if 'TPC' in jdata[i].keys():
            if jdata[i]['TPC'] == 'xroot':
                print(json.dumps(jdata[i]))
            with tpclock:
                if jdata[i]['Xeq']['Type'] == 'pull':
                    ibytes[hostport][instance] += jdata[i]['Size']
                else:
                    obytes[hostport][instance] += jdata[i]['Size']

    msg = "xrootd,host=%s,instance=%s,site=%s,type=tpc ibytes=%d,obytes=%d" % \
          (hostport, instance, sitename, ibytes[hostport][instance], obytes[hostport][instance])
    write2influx(msg, logfile, loglock)

def summaryNetMon(msgroot, logfile, loglock):
    #print("Calling summaryNetMon")
    if msgroot.tag == 'statistics':
        hostport = msgroot.attrib['src']
        instance = msgroot.attrib['ins']
        sitename = msgroot.attrib['site']
    for child in msgroot:
        if child.attrib['id'] == 'link':
            for item in child:
                if item.tag == "in":
                    ibytes = int(item.text)
                elif item.tag == "out":
                    obytes = int(item.text)
    msg = "xrootd,host=%s,instance=%s,site=%s,type=summary ibytes=%d,obytes=%d" % \
          (hostport, instance, sitename, ibytes, obytes)
    write2influx(msg, logfile, loglock)

def write2influx(msg, logfile, loglock):
    t = datetime.datetime.now().timestamp()
    with loglock:
        with open(logfile, "a") as out_file:
            # see https://docs.influxdata.com/influxdb/v2/reference/syntax/line-protocol/
            out_file.write("%s %d\n" % (msg, t*1000000000))
        if ( random.random() > 0.95 ):
            with open(logfile, "r") as in_file:
                lines = in_file.readlines()
            with open(logfile, "w") as out_file:
                for i in range(len(lines)-nlines2keep, len(lines)):
                    if i >= 0:
                        out_file.write(lines[i])

executor = ThreadPoolExecutor(max_workers=2)
while True:
    message, address = serverSocket.recvfrom(65536)
    message = message.decode()

    try:
        # Summary monitoring data is in XML format, while TPC
        # g-Stream detail monitoring data is in JSON format
        msgroot = ET.fromstring(message)
        executor.submit(summaryNetMon, msgroot, logfile, loglock)
    except:
        executor.submit(gstreamTPCMon, message, logfile, loglock, tpclock)
```

## Config Telegraf

Add the following file to /etc/telegraf/telegraf.d/30-xrootd.conf
```
[[inputs.exec]]
    commands = ["cat /tmp/xrdmon.telegraf.log"]
    timeout = "30s"
    data_format = "influx"
```

Now you are ready to go to your Grafana to create a monitoring dashboard. An example Grafana query of InfluxDB 
can be like this:
```
SELECT non_negative_derivative(mean("ibytes"), 1s) FROM "xrootd" WHERE ("instance"::tag = 'atlas' AND "type"::tag = 'summary') AND $timeFilter GROUP BY time($__interval)
```

Multiply Xrootd instances can send the summary monitoring info to the same receiver process. 


