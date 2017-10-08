# Outline

This example describes how to install the NFS server example.

You will create a Web frontend server, an auto-provisioned persistent volume on GCE, and an NFS-backed persistent claim.

This is done using helm (install and initialize helm and tiller first):

* https://docs.helm.sh/using_helm/#install-helm
* https://docs.helm.sh/using_helm/#initialize-helm-and-install-tiller
* https://github.com/kubernetes/helm/releases

```
helm init
kubectl -n kube-system get po
```

tl;dr:

This process has not yet been converted into a helm chart, but will produce
three pods that demonstrate communication across an NFS service.

```
kubectl create -f examples/volumes/nfs/provisioner/nfs-server-gce-pv.yaml
kubectl create -f examples/volumes/nfs/nfs-server-rc.yaml
kubectl create -f examples/volumes/nfs/nfs-server-service.yaml
# get the cluster IP of the server using the following command
kubectl describe services nfs-server
# use the NFS server IP to update nfs-pv.yaml and execute the following
kubectl create -f examples/volumes/nfs/nfs-pv.yaml
kubectl create -f examples/volumes/nfs/nfs-pvc.yaml
# run a fake backend
kubectl create -f examples/volumes/nfs/nfs-busybox-rc.yaml
# get pod name from this command
kubectl get pod -l name=nfs-busybox
# use the pod name to check the test file
kubectl exec nfs-busybox-jdhf3 -- cat /mnt/index.html
```

There are several issues with the current state of this example and some stand
in the way of an easy, direct conversion to helm chart.

Roadmap of todos here, under the quickstart notes:

[ ]: The service IP is not known at the time of instantiating the service, as
reflected by the original non-helm quickstart instructions, the requirement to
run `kubectl describe services nfs-server`.

kingdonb/examples-volumes-nfs#2: You can't use Cluster DNS when setting up PVs

It would be better to depend on cluster discovery or DNS so that the client PV
does not require knowledge of the IP address, instead inferring information
about the location of the service through the known value of the service name.

[ ]: The cloud provider and storage provisioner may be variable and should be
configurable as a user option in values.yaml.

If other storage provisioners require different storageClass settings or
otherwise need adjustments to the provisioner/pv in order to claim a persistent
disk, then these facts should be configurable from the values.yaml.

[ ]: A better example of an NFS consumer than the web rc is possible.

My idea is to build an "off-cluster Minio" service provider for the proposed
change in deis/workflow#857.  The Minio service should be implemented as NFS
clients a-la the current rc-web, each backed by the same ReadWriteMany NFS PV.

The default in Deis Workflow is to start Minio with a memory-backed
storage, so that persistent claims are not needed, but this means the cluster
state will not survive a reboot.  Deis can also use S3/Swift/etc Storage svc
from your cloud host, but there is no provision for a persistent minio server 
or Deis Workflow in a bring-your-own-machines environment.

You could stand up Minio and inject into the Deis environment, as I've done in
[this gist](https://gist.github.com/kingdonb/845945908336cbeefb3e8cce73ec5dc2), 
but without an improved version of something like this example (volumes/nfs)
you have just introduced a new SPOF into your Workflow architecture.

At least it could be safer to scale Minio and back it with an NFS service that
is further backed with a very durable StorageClass driver.

This will never be as resilient as a Ceph service, (that also provides its own
object gateway, obviating need for Minio) but for environments where the extra
overhead of Ceph is not warranted, this could be a more viable alternative.

[ ]: Now that we have built up client, make the server separable.

It should be possible to use the NFS client example against an arbitrary NFS
service.  Add some alternative configuration for an environment where AWS EFS
(for example) is providing the NFS service.

I know there is an EFS driver for PV already, but should be a trivial extension
of this example to connect to EFS or similar and disable the server component.
