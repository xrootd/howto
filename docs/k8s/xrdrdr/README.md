# An Xrootd redirector deployment 

This is a redaced copy of a Xrootd redirector configuration in SLAC's Kubernetes cluster. It runs under
the namespace 'xrdrdr'. In general, anything starting with `my-`, `my_`, `my.` or `xrdrdr` will need to be 
customized based on your operational environment.

## Xrootd redirector configuration

This redirector is intended to work with several data servers that export the same shared file system.
For this reason, the `cms.dfs` directive is used.

## Kubernetes deployment

As a general rule, all Kubernetes infrastructures are not the same. The `services.yaml` here assumes using
a load-balancer [`metallb.universe.tf`](https://metallb.io) (and address pool `my-address-pool`). Your 
k8s infrastructure may use a different kind of load-balancer.

It also expects up-to-dated CAs and VOMS LSCs in a subdir `etc/grid-security/{certificates, vomsdir}` under
PVC `my-pvc`. These CAs and VOMS LSCs will be bind mounted to the containers. So they should be updated
using *in-place* replacement methods - new content of a file overwrites the old content - no new files 
should be created.

This k8s deployment will utilize several secrets (hostcert, hostkey and macaroon) stored in a vault service
`https://vault.my.com`. To upload the secrets to the vault service, run `sh upload-secrets.sh`.

To deploy services, (please pay attend to the kubeconfig file and namespace, and then) run
```
make apply
kubectl --kubeconfig ... -n xrdrdr apply -f configmap.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f services.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f pvc.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f deployment.yaml
```

## Download the example

Download this [tarball](rdr_in_k8s.tar.gz.base64). It contents the Kubernetes yamls, Xrootd Dockerfile and 
Xrootd configuration. Then run
```
cat rdr_in_k8s.tar.gz.base64 | base64 -d > rdr_in_k8s.tar.gz
```

