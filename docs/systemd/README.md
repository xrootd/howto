# Customize Xrootd's Systemd Services

Xrootd releases ship systemd services in `/usr/lib/systemd/system/xrootd@.service` and
`/usr/lib/systemd/system/cmsd@.service` They can be cumstized to fit the operational need.
Below we use the xrootd service as an example. It also applys to the cmsd service. 

To do so, first create a xrootd configuration file /etc/xrootd/xrootd-myinstance.cfg (`myinstance` is the
instance name of xrootd - multiple xrootd can run on the same machine but they need to use different 
TCP ports, and they need to run under unique instance names). Then enable this `myinstance` by running 
```
sudo systemctl enable xrootd@myinstance.service 
```
This will create a file (a symlink) 
`/etc/systemd/system/multi-user.target.requires/xrootd@myinstance.service`.

The service can be customized by creatng a file `/etc/systemd/system/xrootd@myinstance.service.d/local.conf`
with the following content:

```
[Service]
User=userjoe
Group=groupjoe
ExecStart=
ExecStart=/usr/bin/xrootd -l /var/log/xrootd/xrootd.log -c /etc/xrootd/xrootd-%i.cfg -k fifo -s /run/xrootd/xrootd-%i.pid -n %i
WorkingDirectory=/var/spool/xrootd
ExecStartPre=-+/bin/mkdir /run/xrootd
ExecStartPre=+/bin/chmod -R 777 /run/xrootd

[Unit]
After=data-disk1.mount data-disk2.mount
```

The systemd.service man page explains the meaning of these directives. Directives in this customization file
overwrites those in the corresponding system unit file. In this example, the service will be 
started:

   *  After systemd slices `data-disk1.mount` and `data-disk2.mount` are started (The two slices are likely defined base on info in `/etc/fstab` - mount point `/data/disk1` and `/data/disk2` - check with `systemctl list-units`)
   *  First Run those lines in the `ExecStartPre=` lines sequencially (as user root due to the '+' in the lines).
   *  Then Run ExecStart line under user `userjoe`:`groupjoe` instead of the default `xrootd`:`xrootd`. Note that the first `ExecStart=` is to clear the responding `ExecStart=...` line (previously defined) in the systemd unit file.

After creating the customization file, run `sudo systemctl daemon-reload`.



