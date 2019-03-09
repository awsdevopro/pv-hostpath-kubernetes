# pv-hostpath-kubernetes
source: 

https://supergiant.io/blog/persistent-storage-with-persistent-volumes-in-kubernetes/

## Using hostPath Persistent Volumes in Kubernetes

We’ll create a `PersistentVolume`  using `hostPath`  volume plugin and claim it for the use in the Deployment running Apache HTTP servers. `hostPath`  volumes use a file or directory on the Node and are suitable for the development and testing purposes.

Note: `hostPath`  volumes have certain limitations to watch out. It’s not recommended to use them in production. Also, in order for the `hostPath`  to work, we will need to run a single node cluster.  See the official documentation for more info.

To complete this example, we used the following prerequisites:

A Kubernetes cluster deployed with Minikube. Kubernetes version used was 1.10.0.
A kubectl command line tool installed and configured to communicate with the cluster. See how to install kubectl here.

### Step 1 Create a Directory on Your Node
First, let’s create a directory on your Node that will be used by the `hostPath`  Volume. This directory will be a webroot of Apache HTTP server.

First, you need to open a shell to the Node in the cluster. Since you are using Minikube, open a shell by running `minikube ssh` .

In your shell, create a new directory. Use any directory that does not need root permissions (e.g., user’s home folder if you are on Linux):

`mkdir /home/&lt;user-name&gt;/data`
Then create the `index.html`  file in this directory containing a custom greeting from the server (note: use the directory you created):

`echo '<?php phpinfo();' > /home/<user-name>/data/index.php`

### Step 2 Create a Persistent Volume
The next thing we need to do is to create a `hostPath`  PersistentVolume  that will be using this directory.
```shell
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

`kubectl create -f hostpath-pv.yaml`
`persistentvolume "pv-local" created`
Let’s check whether our PersistentVolume was created:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv-local   10Gi       RWO            Retain           Available             local                    2m
```
This response indicates that the  volume is already available but still does not have any claim bound to it, so let’s create one.

### Step 3 Create a PersistentVolumeClaim (PVC) for your PersistentVolume
The next thing we need to do is to claim our PV using a PersistentVolumeClaim . Using this claim, we can request resources from the volume and make them available to our future pods.
```shell
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
`kubectl create -f hostpath-pvc.yaml `
`persistentvolumeclaim "hostpath-pvc" created`

Let’s check the claim running the following command:

`kubectl get pvc hostpath-pvc`
The response should be something like this:
```
NAME           STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
hostpath-pvc   Bound     pv-local   10Gi       RWO            local          29s
```
As you see, our PVC was already bound to the volume of the matching type. Let’s verify that the PV we created was actually selected by the claim:

`kubectl get pv pv-local`
The response should be something like this:
```
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
pv-local   10Gi       RWO            Retain           Bound     default/hostpath-pvc   local    16m
```                
Did you notice the difference from the previous status of our PV? You’ll see that it is now bound by the claim hostpath-pvc  we just created (the claim is living in the default Kubernetes namespace). That’s exactly what we wanted to achieve!

### Step 4 Use the PersistentVolumeClaim as a Volume in your Deployment

Now everything is ready for the use of your hostPath  PV in any pod or deployment of your choice. To do this, we need to create a deployment with a PVC referring to the hostPath  volume. Since we created a PV that mounts a directory with the index.php  file, let’s deploy Apache HTTP server from the Docker Hub repository.

```shell
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: webapp1
spec:
  selector:
    matchLabels:
      app: webapp1
  replicas: 1
  template:
    metadata:
      labels:
        app: webapp1
    spec:
      containers:
      - name: webapp1
        image: awsdevopro/apache-php56
        ports:
        - containerPort: 80
        volumeMounts:
        - name: web
          mountPath: /var/www/html
      volumes:
      - name: web
        persistentVolumeClaim:
          claimName: hostpath-pvc
            

apiVersion: v1
kind: Service
metadata:
  name: webapp1-svc
  labels:
    app: webapp1
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30081
  selector:
    app: webapp1
```
As you see, along with the standard deployment parameters like container image and container port, we have also defined a volume named “web” that uses our PersistentVolumeClaim . This volume will be mounted with our custom index.html  at /var/www/html , which is the default webroot directory of LAMP for this Docker Hub image. Also, deployment will have access to 5Gi of data in the hostPath  volume.

Save this spec in httpd-deployment.yaml  and create the deployment using the following command:

`kubectl create -f httpd-deployment.yaml`
`deployment.apps "httpd" created`

Let’s check the deployment’s details:
```shell
...
Mounts:
      /var/www/html from web (rw)
  Volumes:
   web:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  hostpath-pvc
    ReadOnly:   false
```
Along with other details, the output shows that the directory /var/www/html  was mounted from the web (rw) volume and that our PersistentVolumeClaim  was used to provision the storage.

Now, let’s verify that our LAMP pods actually serve the index.php  file we created in the first step. To do this, let’s first find the UID of one of the pods and get a shell to the Apache server container running in this pod.
`kubectl get pods -l app=webapp1`
```shell
NAME                     READY     STATUS    RESTARTS   AGE
httpd-5958bdc7f5-fg4l5   1/1       Running   0          15m
httpd-5958bdc7f5-jjf9r   1/1       Running   0          15m
```
We’ll enter the httpd-5958bdc7f5-jjf9r  pod using the following command:

`kubectl exec -it httpd-5958bdc7f5-jjf9r -- /bin/bash` 
Now, we are inside the Apache2 container’s filesystem. You may verify this by using Linux `ls`  command.
```
root@httpd-6c96b87dfc-g4r45:/var/www/html# ls
index.php
root@httpd-6c96b87dfc-g4r45:/var/www/html# cat index.php
<?php phpinfo();
```
`curl minikube_ip:30081` to your browser and hit enter. 
Get the PHP versions page and installed all the packages in the image

That’s it! Now you know how to define a PV and a PVC for hostPath volumes and use this storage type in your Kubernetes deployments. This tutorial demonstrated how both PV and PVC take care of the underlying storage infrastructure and filesystem, so users can focus on just how much storage they need for their deployment.


