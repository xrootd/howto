### Avoid launching a DoS attack

XrdEC will create TCP links to all backend server nodes at the same time.
When doing so using `xrdcp` or `xrdfs` in script to process many files, 
a large number of TCP connections will be created and then destroyed in 
short period of time. Some network switches may view this as a DoS attack. 
On the other hand, this is not a problem with xrootd proxy or `xrootdfs`
as them maintain and reuse network connections.

### Patch or upgrade the EC cluster

If restart is needed after patching, or if a new xrootd version is needed,
follow the procedure below:

  1. Restarting a backend redirector or frontend proxy redirector will 
     cause new connections to freeze for a while until the redirector is 
     back. There is no other impact.
  1. To restart a backend data server, first stop the `cmsd` service running
     on the node. Waiting until there is no activities in `xrootd` service 
     (e.g. no TPC connections). then stop the `xrootd` service. Bring both 
     services back afterward.
    * At most `m` number of data servers can be brought offline at the same
      time.
    * The more data servers are brought offline at the same time, the less 
      protect during that period.
    * If there is a `xrootdfs` mounting the backend cluster, avoid using 
      the `xrootdfs` instance during these period. At the minimum, send a
      `SIGUSR1` to the `xrootdfs` process after bring down and bring up the
      `cmsd` on the data servers.
  1. Similarly, once can use the above procedure for proxy servers in the
     proxy cluster.

### What to expect when failures happen

When a data server goes down, all files with corresponding zip files on that
server will more or less be effected. When a disk (real or raided disk) fails, 
all files on the disk will more or less be effected. Base on our observation,
the impact for read and write are different:

  * For users doing a reading from the failed component, expect a freeze of
    length `XRD_CONNECTIONWINDOW * (XRD_CONNECTIONRETRY -1)`
  * For users doing a writing to the failed component, the writing may 
    continue but eventually will failed, with a error message of “connection
    refused” during the writing stage or during the close stage (and rarelly,
    during the opening stage)
  * In case of a server failuer, new write initialized (opened before failure,
    but start writing after the failure) within T seconds after the outage 
    **may** fail if the failed server is used for writing, where
    ```
    T = max (XRD_CONNECTIONWINDOW * (XRD_CONNECTIONRETRY -1), XRD_STREAMERRORWINDOW)
    ```
    - Writing after that period will continue as long as there are at least
      `n + m` data servers.

### How to identify and repair damaged files

Damaged file here referes to files that lost a zip file due to disk or
data server failure. Note that if a EC is configured as `n + m`, then each 
server will at most host one of those `n + m` zip file. This zip file
is stored on one of file systems/disks on that server.

A damaged file has a degradated level of protection. But it is still available 
because there is still `n + m -1` corresponding zip files.

#### Identify damaged files due to a disk failure

Let's first look at the scenario when a disk fails. Suppose a server has:

* two disks `/dev/sda` and `/dev/sdb` mounted at `/disk/sda` and `/disk/sdb`
* The likely xrootd configuration file will have the following lines:

```
all.export /data
oss.space public /disk/sda
oss.space public /disk/sdb
```
In this case, `/data` hosts the name space of all (zip) files storage on 
this server.  Under `/data` is the full directory tree containing symlinks. 
The symlinks point to actual files in `/disk/sda` and `/disk/sdb`. For example 
```
/data/dir1/file1 -> /disk/sdb/public/DA/CA64DA61306C00000000864f8117DA9200000A6%
```
If `/dev/sdb` fails, one can identify all files under `/data` which are symlinks
to `/disk/sdb`. These files will need to be repaired. 

If `/data` is lost, we can recover the name space in `/data` because
each file in `/disk/sda` and `/disk/sdb` actually stores in its extended 
attribute the info of where it belongs to in the name space.
```
$ getfattr -n user.XrdFrm.Pfn /disk/sdb/public/DA/CA64DA61306C00000000864f8117DA9200000A6%
getfattr: Removing leading '/' from absolute path names
# file: disk/sdb/public/DA/CA64DA61306C00000000864f8117DA9200000A6%
user.XrdFrm.Pfn="/data/dir1/file1"
```
By going through all files in `/disk/sda` and `/disk/sdb`, we can reconstruct the 
directory tree under `/data`.

#### Identify damaged files due to a server failure

Server failure should not automatically trigger data repair procedure. This 
is becasue the operation can continue without this failed server. And if the
server failure is not due to disk failuer, no data are lost.

In the rare case when the server and its disks are all lost, one will need to
check all files on other servers, and see which files are missing a zip file.

#### Repair damaged files

With a list of files to be repaired in hand, one can now start the repair 
procedure. This procedure can be summarized as the following steps:

1. identity the files that need to be repaired
2. copy each file to a new name e.g. `myfile.new`
3. (optionally) compare the checksum of the old and new files
4. delete the old file
5. rename the new file to the old name

There are many ways and tools to that can be used for the procedure. The following
describe how to use `xrootdfs` for repairing.

  * mount the EC storage via `xrootdfs` in an host: e.g. 
    `xrootdfs /data -o nworkers=20 -o rdr=root://my_redirector//data -o direct_io`
    - Make sure that the XrdEC configuration file in the 
      [Enabling EC in xrootd clients](#enabling-ec-in-xrootd-clients) is availble.
      The best way to check is to see if you get the correct size of a file in the 
      `xrootdfs` mounted directory tree.
    - If you have 20 data servers, give 20 or a little more to `nworkers`

  * Copying file. One can use any unix command for copying, for example:
    `dd if=myfile of=myfile.new bs=1M iflag=direct oflag=direct`. Copying tools
    that utilize direct IO for input and output, and large block size will perform 
    better. 
  * (optional) valide the the checksum. Usually existing files already have 
    some kind of checksum calculated and stored. For example, if adler32 is used, 
    one can use [`/etc/xrootd/xrdadler32.sh`](#configuring-a-xrootd-proxy-using-ec)
    `myfile` and `/etc/xrootd/xrdadler32.sh myfile.new` to valid the checksum.
  * use 'rm' and 'mv' to delete the old file and rename the new file.

The following script will automate the above steps. It takes a to be repaired file
(on xrootdfs), and repair its
```
$ cat repair.sh
#!/bin/sh

file=$1
CKSM="/etc/xrootd/xrdadler32.sh"  # this is the checksum script mentioned above
nfile="${file}.newcopy"
dd if=$file of=$nfile bs=1M iflag=direct oflag=direct >/dev/null 2>&1
srccksm=$($CKSM $file)
dstcksm=$($CKSM $nfile)
if [ "$srccksm" == "$dstcksm" ]; then
  echo [O] $file
  rm $file
  mv $nfile $file
  exit
else
  echo [X] $file
  rm $nfile
  exit 1
fi
```

### Identify debris left behind

This happens when a data server is offline while a file was deleted, and the 
file was copied back later. XrdEC will automatically ignore these "debris" (zip
files). 

**XrdEC SHOULD print out them for cleaning**: more on this later

### Identify new files for backup

**XrdEC SHOULD log all newly create files**: more on this later
