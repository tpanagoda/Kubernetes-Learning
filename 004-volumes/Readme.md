# Kubernetes Learning - Kubernetes Volumes

<aside>
💡

> Files in this article is available in the Github repository [Kubernetes-Learning.git](https://github.com/tpanagoda/Kubernetes-Learning) refer the folder `002-volumes` to get the required files.
> 
</aside>

## What is a Kubernetes Volume?

A Kubernetes volume provides external storage for containers outside the container file system. Volumes allow data to persist beyond the life cycle of individual containers, enabling data sharing between containers and maintaining state across container restarts.

 

## Key Concepts

### Volumes

Volumes are defined in the pod specification and describe where and how the external data is stored.

### Volume Mounts

Volume mounts are defined in the container specification and attached to the volume (Defined at the pod level) to a specific container within the pod. They also specify the path where the volume data will appear in the container process at runtime.

## Common Volume Types

1. **hostPath:** Stores data in a specific location directly on the host file system (Kubernetes Node)
2. **emptyDir:** Provide temporary storage in an automatically managed location on the host file system.
3. **persistentVolumeClaim:** Mount data stored in a PersistentVolume (We will not cover this type in this article) 

## Real-world Example

When there is a requirement to store the files uploaded by a user in a volume, you can do the following.

1. If the container crashes the data will persist if you store the uploaded files outside the container.
2. You can share the files between different containers (e.g., Webserver and other processes)
3. You can back the file easily without interfering with the application containers.

## Creating Pods with Volumes

## 1. Set up your Kubernetes cluster

You need to have a running Kubernetes Cluster (e.g., minikube, k3 or cloud manager cluster)

Verify the Cluster is running

```yaml
kubectl cluster-info
```

## 2. hostPath Volume

### 2.1 Create a file in the host (Kubernetes Node)

If you have multiple nodes you can create the same folder and the file in each node.

Example:

Worker-001: This is the Worker 001

Worker-002: This is the worker 002

```bash
mkdir /tmp/hostpath
echo 'This is Worker 001' > /tmp/hostpath/hostfile.txt
```

![image.png](assets/image.png)

### 2.2 Create a pod with hostPath volume mounted.

Save the hostPath YAML to a file named `hostpath-volume-pod.yml`

```yaml
## File hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-volume-pod
spec:
  containers:
  - name: my-container
    image: busybox
    command: ["/bin/sh", "-c", "cat /data/hostfile.txt && sleep 3600"]
    volumeMounts:
    - name: host-volume
      mountPath: /data
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/hostpath
      type: Directory

```

### 2.2 Apply the configuration

```bash
kubectl apply -f hostpath-pod.yaml
```

![image.png](assets/image%201.png)

### 2.3 Verify the pod is running

```bash
kubectl get pods
```

![image.png](assets/image%202.png)

### 2.4 Check the pod logs

```bash
kubectl logs hostpath-volume-pod
```

![image.png](assets/image%203.png)

## 3. emptyDir volume

Here, we will create an `emptyDir` volume for temporary data storage.

### 3.1 Create the `emptydir-pod.yaml`  manifest file

```yaml
## File: emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command: ["/bin/sh", "-c", "echo 'Hello from emptyDir' > /cache/data.txt && sleep 3600"]
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: reader
    image: busybox
    command: ["/bin/sh", "-c", "cat /cache/data.txt && sleep 3600"]
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  volumes:
  - name: cache-volume
    emptyDir: {}
```

We will create `emptyDir` volume with the name `cache-volume` and then mount that volume to the path `/cache` in the container `writer` Then create a file `data.txt` in that path with the content `Hello from `emptyDir`  Then create another container with the name `reader` with mounting the same volume to display the content of that file we created.

### 3.2 Apply the configuration

```yaml
kubectl apply -f emptydir-pod.yaml
```

![image.png](assets/image%204.png)

### 3.3 Verify the pod is running

```bash
kubectl get pods
```

![image.png](assets/image%205.png)

### 3.4 Examine the pods

```bash
kubectl logs emptydir-volume-pod -c reader
```

![image.png](assets/image%206.png)

Check whether the data written by the container `writer` is readable by the container `reader`

## 4.4 Clean the setup

Delete the pods

```bash
kubectl delete pods emptydir-volume-pod hostpath-volume-pod
```

![image.png](assets/image%207.png)