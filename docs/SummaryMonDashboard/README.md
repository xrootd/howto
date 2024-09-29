# Setting Up a Summary Monitoring Dashboard in Grafana

The page provides a prototype setting that allow displaying Xrootd summary monitory into on a Grafana
dashboard. It assume that a functioning `Telegraf/InfluxDB/Grafana` system is available. 

## Setting in Xrootd

Add the following line to your Xrootd configuration file:
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

## Run a collector that saves info for Telegraf

The following python script running on 127.0.0.1 will receive the Xrootd summary monitoring info sent out by
the above Xrootd server, calculate the speed of network bytes in and out, and save the data in the 
InfluxDB line protocol format to a file (`/tmp/xrdsummon.telegraf.log`).
```
#!/usr/bin/python3

import socket
import datetime
import xml.etree.ElementTree as ET

serverSocket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
serverSocket.bind(('', 9300))

nlines2keep = 720
logfile = "/tmp/xrdsummon.telegraf.log"

while True:
    message, address = serverSocket.recvfrom(1024)
    root = ET.fromstring(message)
    t = datetime.datetime.now().timestamp()
    if root.tag == 'statistics':
        hostport = root.attrib['src']
        instance = root.attrib['ins']
        sitename = root.attrib['site']
    for child in root:
        if child.attrib['id'] == 'link':
            for item in child:
                if item.tag == "in":
                    ibytes = int(item.text)
                elif item.tag == "out":
                    obytes = int(item.text)
    with open(logfile, "a") as out_file:
        # see https://docs.influxdata.com/influxdb/v2/reference/syntax/line-protocol/
        out_file.write("xrootd,host=%s,instance=%s,site=%s ibytes=%d,obytes=%d %d\n" % 
                       (hostport, instance, sitename, ibytes, obytes, t*1000000000))
    # Trim the log files size in random occasions, in about 10% of the time
    #if ( (t % 1) * 100 > 90 ):
    #    with open(logfile, "r") as in_file:
    #        lines = in_file.readlines()
    #    with open(logfile, "w") as out_file:
    #        for i in range(len(lines)-nlines2keep, len(lines)):
    #            if i >= 0:
    #                out_file.write(lines[i])
```

## Config Telegraf

Add the following file to /etc/telegraf/telegraf.d/30-xrootd.conf
```
[[inputs.exec]]
    commands = ["cat /tmp/xrdsummon.telegraf.log"]
    timeout = "30s"
    data_format = "influx"
```

Now you are ready to go to your Grafana to create a monitoring dashboard. An example Grafana query of InfluxDB 
can be like this:
```
SELECT non_negative_derivative(mean("ibytes"), 1s) FROM "xrootd" WHERE ("instance"::tag = 'atlas') AND $timeFilter GROUP BY time($__interval)
```

In this setup, the data collection frequency is determined by the `xrd.report ... every 60s ...` setting 
in the Xrootd config file. Multiply Xrootd instances can send the summary monitoring info to the same 
receiver process. 


