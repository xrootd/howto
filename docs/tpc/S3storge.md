Xrootd can provide the following services to an S3 object store.

* Object store functions such as Upload/Download/Deletion/Stat/DirListing/Checksum.
* HTTP TPC and xrootd TPC services
* X509/VOMS authentication/authorization (and JWT in the future).

In this case, Xrootd is used as a DTN to the S3 storage. It authenticates to the 
S3 storage using a pair of access key and secret key. Configuring xrootd in this 
use case is largely based on  the [WLCG TPC configurat example](#an-example-of-wlcg-tpc-configuration-with-x509-authentication), with the following additional configurations:

  * S3 object store uses the word **bucket** for **directory**
  * Change to the configuration file (the `server` / `proxy serve` section in the `else` clause) <p>
  ```
  pss.origin https://s3store.xyz:port
  setenv AWS_ACCESS_KEY_ID = GOOGS4MV4PQTK3D4G0LB34AN
  setenv AWS_SECRET_ACCESS_KEY = QtUMprl2j07m4VfzygcjJQlVrvO9ZfU+eKDbR2sP
  ```
  * It will use scripts for checksum and xrootd TPC (examples below).
    - in the `else` clause of the configuration file, change the adler32 line to <p>
    ```
    xrootd.chksum max 32 adler32 /etc/xrootd/xrdadler32.sh
    ``` 
  * It needs the Davix rpm (available from EPEL)
  * A staging space. When writing to s3 storage, Davix temporarily store a complete file 
    to this staging area before it uploads the file to the s3 store.
    - The default space is /tmp. It can be changed by unix environment variable 
      **DAVIX_STAGING_AREA**
  * It needs the XrdClHttp plugin and configuration to load the plugin

### Checksum and an example /etc/xrootd/xrdadler32.sh

```
#!/bin/sh

file=$1
davix-get --s3accesskey $AWS_ACCESS_KEY_ID \
          --s3secretkey $AWS_SECRET_ACCESS_KEY \
          --s3alternate https://${XRDXROOTD_PROXY}$file | xrdadler32 | awk '{print $1}'
```

As you see from the above script, the ckecksum is calculated by fetch (`davix-get`) the
file from the s3 storage. There are two drawbacks:

* no place to store the ckecksum
* may incure Egress charge if the s3 store is in a commercial cloud, e.g. 
  [Google using HMAC key](https://cloud.google.com/storage/docs/authentication/hmackeys)
  or AWS, but xrootd runs outside of the cloud.

### An example /etc/xrootd/xrdcp-tpc.sh

```
#!/bin/sh
set -- `getopt S: -S 1 $*`
while [ $# -gt 0 ]
do
  case $1 in
  -S)
      ((nstreams=$2-1))
      [ $nstreams -ge 1 ] && TCPstreamOpts="-S $nstreams"
      shift 2
      ;;
  --)
      shift
      break
      ;;
  esac
done

src=$1
dst=$2
xrdcp --server -s $TCPstreamOpts $src - | \
  aws --endpoint-url https://$XRDXROOTD_PROXY s3 cp - s3:/$dst 2>/dev/null
```

In the script, `aws` is the [AWS Command Line Interface](https://aws.amazon.com/cli/). Loading `aws`
to run may take a second or two. If this is too slow, one can use `davix-put`. However, using 
`davix-put` requires the `xrdcp` to first save the file - `davix-put` does not read from STDIN.

### The XrdClHttp plugin

The `XrdClHttp` plugin (`libXrdClHttp-5.so` or `libXrdClHttp.so`) is so far not available in xrootd
rpms. One needs to [compile xrootd](../Compile) to create this plugin `.so`. After this plugin is
created, copy it to /usr/lib64 and then run `chrpath -d /usr/libi64/libXrdClHttp.so`.

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
