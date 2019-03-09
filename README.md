# pv-hostpath-kubernetes
source: 

https://supergiant.io/blog/persistent-storage-with-persistent-volumes-in-kubernetes/

we’ll create a `PersistentVolume`  using `hostPath`  volume plugin and claim it for the use in the Deployment running Apache HTTP servers. `hostPath`  volumes use a file or directory on the Node and are suitable for the development and testing purposes.

Note: `hostPath`  volumes have certain limitations to watch out. It’s not recommended to use them in production. Also, in order for the `hostPath`  to work, we will need to run a single node cluster.  See the official documentation for more info.

To complete this example, we used the following prerequisites:

A Kubernetes cluster deployed with Minikube. Kubernetes version used was 1.10.0.
A kubectl command line tool installed and configured to communicate with the cluster. See how to install kubectl here.

Step #1 Create a Directory on Your Node
First, let’s create a directory on your Node that will be used by the `hostPath`  Volume. This directory will be a webroot of Apache HTTP server.

First, you need to open a shell to the Node in the cluster. Since you are using Minikube, open a shell by running `minikube ssh` .

In your shell, create a new directory. Use any directory that does not need root permissions (e.g., user’s home folder if you are on Linux):

1
`mkdir /home/&lt;user-name&gt;/data`
Then create the `index.html`  file in this directory containing a custom greeting from the server (note: use the directory you created):

1

`echo 'Hello from the hostPath PersistentVolume!' > /home/<user-name>/data/index.html`

###Step #2 Create a Persistent Volume
The next thing we need to do is to create a `hostPath`  PersistentVolume  that will be using this directory.
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-local
  labels:
    type: local
spec:
  storageClassName: local
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/<user-name>/data"
```

This spec:

defines a PV named “pv-local” with a 10Gi capacity ( spec.capacity.storage ).
sets the PV’s access mode to ReadWriteOnce, which allows the volume to be mounted as read-write by a single node ( spec.accessModes ).
- assigns a storageClassName  “local” to the PersistentVolume .
- configures hostPath  Volume plugin to mount local directory at /home/<user-name>/data
- Save this spec in the file (e.g hostpath-pv.yaml ) and create the PV running the following command:

# kubectl create -f hostpath-pv.yaml
`persistentvolume "pv-local" created`
# Let’s check whether our PersistentVolume was created:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv-local   10Gi       RWO            Retain           Available             local                    2m
```
This response indicates that the  volume is already available but still does not have any claim bound to it, so let’s create one.

Step #3 Create a PersistentVolumeClaim (PVC) for your PersistentVolume
The next thing we need to do is to claim our PV using a PersistentVolumeClaim . Using this claim, we can request resources from the volume and make them available to our future pods.
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: hostpath-pvc
spec:
  storageClassName: local
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      type: local
```
Our PVC does the following:

filters the volumes labeled “local” to bind our specific hostPath  volume and other hostPath  volumes that might be created later ( spec.selector.matchLabels ) .
- targets hostPath  volumes that have ReadWriteOnce  access mode ( spec.accessModes ).
- requests a volume of at least 5Gi ( spec.resources.requests.storage ).
- First, save this resource definition in the hostpath-pvc.yaml , and then create it similar to what we did with the PV:
# kubectl create -f hostpath-pvc.yaml 
`persistentvolumeclaim "hostpath-pvc" created`

Let’s check the claim running the following command:
```
kubectl get pvc hostpath-pvc
The response should be something like this:

NAME           STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
hostpath-pvc   Bound     pv-local   10Gi       RWO            local          29s
As you see, our PVC was already bound to the volume of the matching type. Let’s verify that the PV we created was actually selected by the claim:

kubectl get pv pv-local
The response should be something like this:

NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
pv-local   10Gi       RWO            Retain           Bound     default/hostpath-pvc   local                    16m
Did you notice the difference from the previous status of our PV? You’ll see that it is now bound by the claim hostpath-pvc  we just created (the claim is living in the default Kubernetes namespace). That’s exactly what we wanted to achieve!
```
Step #4 Use the PersistentVolumeClaim as a Volume in your Deployment
Now everything is ready for the use of your hostPath  PV in any pod or deployment of your choice. To do this, we need to create a deployment with a PVC referring to the hostPath  volume. Since we created a PV that mounts a directory with the index.html  file, let’s deploy Apache HTTP server from the Docker Hub repository.





