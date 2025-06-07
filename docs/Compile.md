# How to compile xrootd

Make sure your environment does not have xrootd rpms installed.

## Environment requirement

Build an AlmaLinux 9 Apptainer container to compile Xrootd, tested on Rocky Linux 8 (*experimental*)

```
# save this file as xrdbld.almalinux9.def
  
# How to build apptainer image (sif and sandbox) as non-root user:
# 1. build a .sif image
#   apptainer build xrdbld.$(date +%Y%m%d).almalinux9.sif xrdbld.almalinux9.def
# 2. build sandbox (expanded directory)
#   2.1 check your uid/gid are in /etc/subuid and /etc/subgid
#   2.2 apptainer build --fakeroot --sandbox xrdbld.$(date +%Y%m%d).almalinux9 xrdbld.almalinux9.def
# 3. modify sandbox (install other rpms)
#   apptainer shell --fakeroot -w xrdbld.$(date +%Y%m%d).almalinux9

BootStrap: docker
From: almalinux:9

%setup

%post
  dnf install -y 'dnf-command(config-manager)'
  dnf config-manager --set-enabled crb
  dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
  dnf install -y expect perl policycoreutils selinux-policy
  dnf install -y readline-devel libxml2-devel python3-devel
 #dnf install -y curl libcurl-devel
  dnf install -y libmacaroons libmacaroons-devel json-c json-c-devel uuid libuuid-devel
  dnf install -y openssl-devel davix-libs davix-devel voms voms-devel fuse fuse-devel
  dnf install -y scitokens-cpp scitokens-cpp-devel
  dnf install -y git cmake cmake3 make gcc gcc-c++ gdb
  dnf install -y autoconf automake libtool libasan

  # Install gfal2, FTS clients, Rucio clients
 #dnf install -y gfal2-util-scripts gfal2-python3 gfal2-plugin-file gfal2-plugin-http python3-gfal2-util
 #dnf install -y fts-rest-cli 
 #python3 -m pip install -U pip
 #pip3 install rucio-clients

  # To compile FTS3 (then, in fts3 git repo, 'cmake -DALLBUILD=ON')
 #dnf install -y boost-devel glib2-devel libdirq-devel mysql-devel soci-mysql-devel \
 #               activemq-cpp-devel gfal2-devel jsoncpp-devel cryptopp-devel cppzmq-devel \
 #               gridsite-devel globus-gsi-credential-devel protobuf-devel

  dnf update -y
  dnf clean all

%runscript
```

On CentOS 8, the following rpms are needed to compile Xrootd. Some are available from EPEL.

* yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
* yum install -y expect perl policycoreutils selinux-policy 
* yum install -y readline-devel libxml2-devel python2-devel python39-devel
* yum install -y curl libcurl-devel libmacaroons libmacaroons-devel json-c json-c-devel uuid libuuid-devel
* yum install -y openssl-devel davix-libs davix-devel voms voms-devel fuse fuse-devel
* yum install -y git cmake cmake3 make gcc gcc-c++ 
* yum install -y autoconf automake libtool libasan

On CentOS 7, the following rpms are needed to compile Xrootd. Some are available from EPEL.

* yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
* yum install -y expect perl policycoreutils selinux-policy
* yum install -y readline-devel libxml2-devel python-devel python3-devel
* yum install -y curl libcurl-devel libmacaroons libmacaroons-devel json-c json-c-devel uuid libuuid-devel
* yum install -y openssl-devel davix-libs davix-devel voms voms-devel fuse fuse-devel 
* yum install -y scitokens-cpp scitokens-cpp-devel
* yum install -y git cmake cmake3 make gcc gcc-c++
* yum install -y autoconf automake libtool yasm help2man 
* yum install -y centos-release-scl
* yum install -y devtoolset-7

## Instruction to compile xrootd

### Download from github.com

```
basedir=$(pwd)
git clone git@github.com:xrootd/xrootd
cd xrootd
```

If you need `XrdClHttp` (to support Xcache on HTTP, or to support `s3` storage), or `XrdCeph`, do

```
git submodule init
git submodule update -- src/XrdClHttp
git submodule update -- src/XrdCeph
```

If you need the latest `XrdClHttp`, do

```
cd src
git clone git@github.com:xrootd/xrdcl-http
mv XrdClHttp XrdClHttp.save
mv xrdcl-http XrdClHttp
cd ..
```

### Compile

```
cd $basedir
scl enable devtoolset-7 sh # CentOS 7 only
mkdir build
cd build
cmake3 -DCMAKE_INSTALL_PREFIX=. ../xrootd
make -j
[ ! -z "$X_SCLS" ] && exit # CentOS 7 only
cd $basedir
```

Add the following option to cmake3 if you need to:

* support VOMS: `-DENABLE_VOMS=True`
* support HTTP TPC: `-DENABLE_VOMS=True -DBUILD_MACAROONS=1`
* support XrdClHTTP: `-DXRDCLHTTP_SUBMODULE=1` (built by default, no longer needed)
* support Erasure Coding: `-DENABLE_XRDEC=True`
* support ASAN (CentOS 8 only): `-DENABLE_ASAN=True`

### Use the compiled xrootd

create a setup script in `$basedir`

```
cd $basedir
cat > setup.sh <<EOF
#!/bin/sh

me=$(readlink -f $BASH_SOURCE)
mydir=$(dirname $me)

if [ -z "$1" ]; then
  echo "which build?"
  false
else
  bld=$1
  export xrddir=$mydir/$bld/src
  if [ -d $xrddir ]; then
    export PATH=$PATH:$xrddir:$xrddir/XrdCl
    export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$xrddir:$xrddir/XrdCl:$xrddir/XrdClHttp/src:$xrddir/XrdEc
  else
    echo "xrddir $xrddir not found"
    false
  fi
fi
EOF
chmod 755 setup.sh
```

Finally, run `. ./setup.sh build` to setup the xrootd environment.
