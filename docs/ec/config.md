### The backend xrootd storage cluster

Since the backend xrootd cluster isn't aware of EC, there is no extra xrootd 
configuration other than setting up a plain xrootd cluster using a relatively new
xrootd release (e.g. xrootd 5.3.0+) . There is a few things to consider for HW 
configuration:

1. The xrootd cluster needs at least ** n + m ** data servers.
2. EC already provides data redundancy. This reduces the need of using RAID. If
   RAID is not used, one can xrootd's 
   [`oss.space` directive](https://xrootd.slac.stanford.edu/doc/dev54/ofs_config.htm#_Toc89982407)
   to make indivudual filesystems available in xrootd. This method actually
   provides an extra benefit when identifying miss files due to hardware failure.

### Enabling EC in xrootd clients

Xrootd clients that directly facing the backend storage will need to load the XrdEC 
library. These clients include adminsitrative tools `xrdcp/xrdfs/xrootdfs` and user 
facing xrootd proxy. To enable EC in xrootd clients:

1. use a xrootd release that includes `libXrdEC.so` (likely xrootd 5.4.1).
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
Here `nbdta` refers to the number of data chunks in an EC stripe, and `nbprt` refers
to the number of parity chunks in the stripe. `chsz` referst to the chunk size (Bytes).

The extension of the file must be **.conf**.

**Tips**

1. Even with XrdEC in the setting, you may want to access an individual xrootd data server 
   to check something. In that case, use the *xroot://...* protocol because the above 
   config file won't load XrdEC except for *root://...* protocol.
2. If you can't add the above config file, you can use unix env. var. XRD_PLUGINCONFDIR
   to point to the directory that holds this file.

### Configuring a xrootd proxy using EC.

In addition to the requirement in the [Enabling EC in xrootd clients](#enable-ec-in-xrootd-clients) 
section, please refers to the 
[WLCG TPC example configuration](../tpc/#an-example-of-wlcg-tpc-configuration-with-x509-authentication)
document for xrootd proxy (DTN) configuration. On the proxy server, an external checksum script
is needed (using the following directive in xrootd configuration file).
```
xrootd.chksum adler32 /etc/xrootd/xrdadler32.sh
```

#### An example of **/etc/xrootd/xrdadler32.sh**

```
#!/bin/sh

# Make sure XrdEC is loaded, and there is no security issue.
# Otherwise this script will not give the correct result.

cksmtype="adler32"
file=$1

# For testing purpose: manually supply XRDXROOTD_PROXY (format: host:port)
# Note: when used by xrootd proxy, XRDXROOTD_PROXY is aleady defined.
[ -z "$2" ] || export XRDXROOTD_PROXY=$2

# If this is a xrootdfs path, try getting XRDXROOTD_PROXY that way.
if [ -z "$XRDXROOTD_PROXY" ]; then
    file=$(getfattr -n xroot.url --only-value $1 2>/dev/null)

    # too bad, this is not a xrootdfs path
    [[ $file = root://* ]] || exit 1

    XRDXROOTD_PROXY=$(echo $file | sed -e 's+^root://++g' | awk -F\/ '{print $1}')
    file=$(echo $file | sed -e "s+^root://++g; s+^$XRDXROOTD_PROXY++g")
fi

hostlist=""
strpver=0
for h in $(xrdfs $XRDXROOTD_PROXY locate -h "*" | awk '{print $1}'); do
    ver=$(xrdfs $h xattr $file get xrdec.strpver 2>/dev/null | tail -1 | awk -F\" '{print $2}')
    [ -z "$ver" ] && continue          # this host does not have the strpver, or does not have the file
    [ $ver -lt $strpver ] && continue  # this host has an older strpver, ignore! 
    if [ $ver -eq $strpver ]; then
        hostlist="$hostlist $h"
    else 
        strpver=$ver
        hostlist="$h"
        cksm=$(xrdfs $h xattr $file get xrdec.${cksmtype} 2>/dev/null | tail -1 | awk -F\" '{print $2}')
    fi  
done

if [ -z "$cksm" ]; then
    cksm=$(xrdcp -C ${cksmtype}:print -s -f root://$XRDXROOTD_PROXY/$1 /dev/null 2>&1 | \
         head -1 | \
         awk '{printf("%8s",$2)}' | \
         sed -e 's/\ /0/g')
    for h in $hostlist; do
        xrdfs $h xattr $file set xrdec.${cksmtype}=$cksm 
    done
fi
echo $cksm
```
Do not forget to `chmod +x /etc/xrootd/xrdadler32.sh`. This script can also be used by 
`xrootdfs` to verify checksum in an XrdEC storage.

#### An example of **/etc/xrootd/xrdcp-tcp.sh**

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

xrdcp --server -s $TCPstreamOpts $src - | xrdcp -s - root://$XRDXROOTD_PROXY/$dst
```
Do not forget to `chmod +x /etc/xrootd/xrdcp-tpc.sh`.

