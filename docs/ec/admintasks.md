### Avoid launching a DoS attack

XrdEC will create TCP links to all backend server nodes at the same time.
When doing so using `xrdcp` or `xrdfs`, a large number of TCP connections will be created and then
destroyed in short period of time. Some network switches may view this as a DoS attack. This is not
a problem with xrootd proxy or `xrootdfs` as them maintain and reuse network connections.

More comming later
