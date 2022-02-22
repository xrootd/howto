### Administrative tasks

One thing to consider is that XrdEC will create links to all backend server nodes at the same time.
When doing so using `xrdcp` or `xrdfs`, large number of TCP connections will be created and then
destroyed in short period of time. Some network switches may view this as a DoS attach. This is not
a problem for xrootd proxy or for `xrootdfs` as them maintain and reuse network connections.

More comming later
