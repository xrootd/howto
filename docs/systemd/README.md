# Xrootd Systemd Service Customization Guide
This document describes how to customize Xrootd's systemd services to fit specific operational needs. There are 
two ways to customize the services: customizing the systemd services with sudo privilegs, and run systemd services
completed from user's home directory without sudo privilegs.

## Customize Xrootd's systemd services with sudo privilegs

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

## Running Xrootd's systemd services from user's home directory without sudo privilegs

Xrootd's systemd services can also be run from user's home directory without sudo privilegs. In the example below, 
we place xrootd configruation file under $HOME/etc/xrootd, and other administrative and log files under $HOME/var
so as to make the service completely under user's control. 

### Step 1: Create necessary directories

Create the following directories:

   * $HOME/etc/xrootd
   * $HOME/var/spool/xrootd $HOME/var/log/xrootd $HOME/var/run/xrootd
   * $HOME/.config/systemd/user

### Step 2: Create Xrootd configuration file

Create a xrootd configuration file ~/etc/xrootd/xrootd-myinstance.cfg (`myinstance` is the instance name of xrootd).
The following is a simple example that illustrates the necessary settings. Adjust the settings to fit your need.

```
all.sitename MYSITE
set home = $HOME
all.export /data  # adjust to your data directory

acc.audit deny
acc.authdb $(home)/etc/xrootd/auth_file
acc.authrefresh 60
ofs.authorize 1
```

Avoid using `all.adminpath`, `all.pidpath`, `all.homepath`, etc. in the xrootd configuration file, as these paths
will be set in the systemd service unit file (below).

### Step 3: Create systemd service file

Create a systemd service file `$HOME/.config/systemd/user/xrootd@.service` with the following content:

```
[Unit]
Description=XRootD xrootd daemon instance %I
Documentation=man:xrootd(8)
Documentation=http://xrootd.org/docs.html

[Service]
ExecStart=/usr/bin/xrootd -c %h/etc/xrootd/xrootd-%i.cfg \
                          -l %h/var/log/xrootd/xrootd.log \
                          -s %h/var/run/xrootd/xrootd-%i.pid \
                          -a %h/var/spool/xrootd \
                          -w %h/var/spool/xrootd \
                          -k 7 -n %i
Type=simple
Restart=on-abort
RestartSec=10
KillMode=control-group
LimitNOFILE=65536
WorkingDirectory=/var/spool/xrootd

[Install]
RequiredBy=default.target
```

Notes:

   * Do not use The `User=` and `Group=` lines in the [Service] section.
   * The log file and config file paths are adjusted to point to user's home directory. 
   * In systemd service unit files, use `%h` instead of `$HOME`.

### Step 4: Enable and start the service

The following commands can be used to enalbe/disable/start/stop, etc.

   * `systemctl --user enable xrootd@myinstance`. This command will use `xrootd@myinstance.service` file
first if it exists. If it does not exist, it will use `xrootd@.service` file to create the service instance.
   * `systemctl --user start xrootd@myinstance`
   * `systemctl --user status xrootd@myinstance`
   * `systemctl --user stop xrootd@myinstance`
   * `journalctl --user -u xrootd@myinstance` to check the log messages of the service (if you are in group
`systemd-journal`).
   * `systemctl --user disable xrootd@myinstance`

To make the user systemd services start automatically at user login (and persist through logout and system
reboot), run the following command (once):

```
loginctl enable-linger
```
