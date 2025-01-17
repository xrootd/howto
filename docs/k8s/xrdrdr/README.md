# An Xrootd redirector deployment 

This is a redaced copy of a Xrootd redirector configuration in SLAC's Kubernetes cluster. It runs under
namespace 'xrdrdr'. In general, anything started with `my-`, `my_`, `my.` or `xrdrdr` will need to be 
customized based on your operational environment.

## Xrootd redirector configuration

This redirector is intended to work with several data servers that export the same shared file system.
For this reason, the `cms.dfs` directive is used.

This configuraiton will utilize several secrets (hostcert, hostkey and macaroon) stored in a vault service
`https://vault.my.com`.
It also expects up-to-dated CAs and VOMS LSCs in a subdir `etc/grid-security/{certificates, vomsdir}` under
PVC `my-pvc`. (Note that these CAs and VOMS LSCs are bind mounted to the containers. So they should be updated
using *in-place* replacement methods - new content of a file overwrites the old content - no new files 
should be created)

## Deployment

To upload secrets (hostcert, hostkey and macaroon) to the vault service, run `sh upload-secrets.sh`.

To deploy services, (please pay attend to the kubeconfig file and namespace, and then) run
```
make apply
kubectl --kubeconfig ... -n xrdrdr apply -f configmap.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f services.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f pvc.yaml
kubectl --kubeconfig ... -n xrdrdr apply -f deployment.yaml
```

## Download

Download this [Tarball](rdr_in_k8s.tar.gz.base64). It contents the Kubernetes yamls, Xrootd Dockerfile and 
Xrootd configuration. Then run
```
cat rdr_in_k8s.tar.gz.base64 | base64 -d > rdr_in_k8s.tar.gz
```

