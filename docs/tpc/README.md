# How to configure Xrootd for Third Party Copy (TPC)

When configurating Xrootd for 3rd Parth Copy, there are several things to consider:

1. Is this a standard along Xrootd or a cluster of Xrootd ?
2. Is the storage accessible to the Xrootd server as a file system? e.g. ext4, xfs, GPFS, Lustre?
3. Is the storage accsssible to the Xrootd server as a remote object store?
4. What data transfer protocol(s) will be used, xroot and/or https?
5. What authentication/authoriaztion mechanism will be used? X509/VOMS, Token, sss key?

There is no way to cover all the above combinations but we will provide examples to cover some typical use cases. 
We will assume that `TLS` is always enabled (which means using Xrootd release `5.3.0` and above).

## A simple TPC using xroot protocol and rendezvous key
{% include-markdown "./SimpleXrootTPCwRendezvousKey.md" %}

## An example of WLCG TPC configuration with X509 authentication
{% include-markdown "./WLCGx509TPC.md" %}
