Xrootd can provide the following services to an S3 object store.

* Object store functions such as Upload/Download/Deletion/Stat/DirListing/Checksum.
* HTTP TPC and xrootd TPC services
* X509/VOMS authentication/authorization (and JWT in the future).

In this case, Xrootd is used as a DTN to the S3 storage. It authenticates to the 
S3 storage using a pair of access key and secret key (aka, username/password).
Configuring xrootd in this use case is largely based on  the 
[WLCG TPC configurat example](#an-example-of-wlcg-tpc-configuration-with-x509-authentication),
with the following additional configurations:

  * S3 object store uses the word **bucket** for what we usually called **top level directory**.
  * Change to the configuration file (the `server` / `proxy serve` section in the `else` clause)

    ```
    pss.origin https://s3store.xyz:port
    setenv AWS_ACCESS_KEY_ID = GOOGS4MV4PQTK3D4G0LB34AN
    setenv AWS_SECRET_ACCESS_KEY = QtUMprl2j07m4VfzygcjJQlVrvO9ZfU+eKDbR2sP
    ```

    (Note that to demostrate the idea, we leave the credentials in the xrootd config file. 
    <b>This is not a good security practice</b>. `setenv` directive allows 
    [putting these credentials in files](https://xrootd.slac.stanford.edu/doc/dev55/Syntax_config.htm#_Toc520499869).
    The variables set by `setenv` are Unix environment variables so they can also be set
    accordingly before xrootd starts.)

  * Disabling Davix session caching to prevent memory leak and fragmentation
    ```
    setenv DAVIX_DISABLE_SESSION_CACHING = 1
    ```

  * xrootd will use scripts for checksum and xrootd TPC (examples below).
    - in the `else` clause of the configuration file, change the adler32 line to <p>
    ```
    xrootd.chksum max 32 adler32 /etc/xrootd/xrdadler32.sh
    ``` 
  * A staging space. When writing to s3 storage, Davix temporarily stores a complete file 
    to this staging area before it uploads the file to the s3 storage.
    - The default space is /tmp. It can be changed by unix environment variable 
      **DAVIX_STAGING_AREA**
    - As of Davix 0.8.7, this area is no longer needed if the following is defined:
    ```
    setenv DAVPOSIX_MPUPLOAD = 1
    ```
  * xrootd needs the XrdClHttp plugin and configuration to load the plugin. The rpm for this
    plugin (and the configuration file) is available in EPEL as `xrdcl-http`
  * Install the Davix rpm (available from EPEL).

### Checksum and an example /etc/xrootd/xrdadler32.sh

```
#!/bin/sh

file=$1
davix-get --s3accesskey $AWS_ACCESS_KEY_ID \
          --s3secretkey $AWS_SECRET_ACCESS_KEY \
          --s3alternate https://${XRDXROOTD_PROXYURL}$file | xrdadler32 | awk '{print $1}'
```

As you see from the above script, the ckecksum is calculated by fetching (using `davix-get`)
the file from the s3 storage. There are two drawbacks:

* no place to store the ckecksum
* may incure Egress charge if the s3 store is in a commercial cloud, e.g. 
  [Google (using HMAC key)](https://cloud.google.com/storage/docs/authentication/hmackeys)
  or AWS, but xrootd runs outside of the cloud.

### An example /etc/xrootd/xrdcp-tpc.sh

```
#!/bin/sh

src=$1
dst=$2
xrdcp - | aws --endpoint-url https://$XRDXROOTD_PROXYURL s3 cp - s3:/$dst 2>/dev/null
```

In the script, `aws` is the [AWS Command Line Interface](https://aws.amazon.com/cli/). Loading `aws`
to run may take a second or two. If this is too slow, one can use `davix-put`. However, using 
`davix-put` requires the `xrdcp` to first save the file - `davix-put` does not read from STDIN.

### The XrdClHttp plugin

This plugin is available from the EPEL repo as the 'xrdcl-http' rpm. The following is for those who 
needs the lastest updates that is not available in the rpm. This is no common.

If the latest feature in the `XrdClHttp` plugin (`libXrdClHttp-5.so` or `libXrdClHttp.so`) is not 
available in the above rpm, one will need to [compile xrootd](../Compile) to create this plugin.
After that copy the `.so` to /usr/lib64 and then run `chrpath -d /usr/libi64/libXrdClHttp.so`.

To instruction xrootd to load this plugin, create `/etc/xrootd/client.plugins.d/xrdcl-http-plugin.conf`
with the following lines inside:
```
url = https://*;http://*
lib = /usr/lib64/libXrdClHttp.so
enable = true
```

### Renaming 

As a final note, **renaming** is poorly supported in s3 object store. For this reason, the 
`XrdClHttp` plugin disabled renaming.

### A complete example

On an AlmaLinux 9 machine, one will need the following rpms:

```
expect perl policycoreutils selinux-policy logrotate libunwind graphviz gv 
libmacaroons voms scitokens-cpp davix python3 python3-boto3

xrootd xrootd-libs xrootd-server xrootd-server-libs xrootd-client xrootd-client-libs 
xrootd-selinux xrootd-voms xrootd-scitokens xrdcl-http
```

For the `davix` rpm, make sure it is `0.8.7` or above.

Host certificate should be available at `/etc/grid-security/xrd/{xrdcert.pem, xrdkey.pem}` with 
correct owership and permission (644 and 600). `/etc/grid-security/{certificates, vomsdir}` are also 
needed

This following directory tree containers a complete example setup that usually goes to /etc/xrootd. 
<code>
├── etc
│   └── xrootd
│       ├── client.plugins.d
│       │   └── [xrdcl-http-plugin.conf](s3_example/client.plugins.d/xrdcl-http-plugin.conf.txt)
│       ├── [robots.txt](s3_example/robots.txt)
│       ├── [cksm4s3.py](s3_example/cksm4s3.py.txt)
│       ├── [xrdcp-tpc.sh](s3_example/xrdcp-tpc.sh.txt)
│       ├── macaroon-secret: [This file needs to be created](../tpc/#generating-the-macaroon-secret)
│       ├── [auth_file](s3_example/auth_file.txt)
│       ├── [scitokens.cfg](s3_example/scitokens.cfg.txt)
│       └── [xrootd.cfg](s3_example/xrootd.cfg.txt)
</code>

To start with this example, one will need to at least replace the following in the last three files:

* The xrootd node name `dtn.slac.stanford.edu` (in `scitokens.cfg`)
* The backend s3 node name `s3.slac.stanford.edu` (in `xrootd.cfg`)
* The storage path (s3 bucket/subbucket) `/xrootd/atlas` (in all three files at the bottom)

The example above uses a different xrdcp-tpc script and a different adler32 script than those in the
previous sections.
