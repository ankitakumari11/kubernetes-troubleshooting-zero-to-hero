### 📌 Stateful and Stateless application
A stateful application remembers past interactions, storing session data on the server to process future requests. A stateless application treats each request as an independent transaction, with all necessary information contained within the request itself, often using client-side data like cookies. This makes stateless applications easier to scale and deploy across different servers, while stateful applications require more complex management to maintain user session continuity across a distributed system.  

### 📌 What is a StatefulSet in Kubernetes?
  
A StatefulSet is a Kubernetes workload object used to manage stateful applications — applications that need:
- ✔ Stable network identity
- ✔ Stable persistent storage
- ✔ Ordered deployment, scaling, and deletion  
StatefulSets guarantee that each pod gets a persistent, unique identity even after restarts.  

**🚀 Why Deployment is not enough for Stateful apps**
  
A Deployment is great for stateless apps, but it does NOT guarantee:  
- ❌ stable pod names
- ❌ stable storage
- ❌ ordering (start/stop sequence)
- ❌ sticky identity

**🧱 Key Features of StatefulSet**
- 1️⃣ Stable, unique pod names : Pod names follow a predictable format.
  - Even if pod 0 is deleted, it comes back as mypod-0, never as a random name.
```
mypod-0
mypod-1
mypod-2
```
- 2️⃣ Stable Persistent Volumes
  - Each pod gets its own PVC created automatically:
  - These volumes are not deleted when the pod is deleted.
```
mypod-0 → pvc-mypod-0
mypod-1 → pvc-mypod-1
```
- 3️⃣ Ordered Deployment & Scaling
  - By default: Pods start in order:0 → 1 → 2
  - Pods terminate in reverse order: 2 → 1 → 0
  - This is ideal for clustered databases.

- 4️⃣ Consistent identity through rescheduling
  - Even if a node crashes or pod restarts, Kubernetes ensures:
    - Same pod name
    - Same hostname
    - Same volume
  
| Feature        | Deployment     | StatefulSet           |
| -------------- | -------------- | --------------------- |
| Pod Names      | Random         | Fixed (pod-0, pod-1…) |
| Storage        | Shared/Generic | Per-pod unique PVC    |
| Pod Identity   | Not preserved  | Always preserved      |
| Startup Order  | Not guaranteed | Guaranteed (0,1,2…)   |
| Shutdown Order | Not guaranteed | Guaranteed (reverse)  |
| Use Case       | Stateless apps | Stateful apps         |
| Examples       | Web apps, APIs | DBs, Kafka, Zookeeper |

## StatefulSet with Persistent Volume not working after Cloud Migration | How to fix it ?  

**Scenario**
- There's a developer who wrote a statefulset menifest file for his database application.
- That yml file ran successfully on aws cloud provider and pods were running successfully.
- But when he tried to run it on Minikube or GCP or Azure, he faced issue as the pods were not coming up.
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  minReadySeconds: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ebs
      resources:
        requests:
          storage: 1Gi
```
<img width="769" height="179" alt="image" src="https://github.com/user-attachments/assets/526cb6e6-7b45-4af8-b261-18f973b29c9d" />
  
- Now here u can see , we had written 3 replicas but it;s showing only 1 becoz in statefulsets once the 1st replica gets created then 2nd starts then 3rd and so on unlike deployments.  
<img width="963" height="299" alt="image" src="https://github.com/user-attachments/assets/baec3d94-ea7f-4509-b776-88b1e56ee694" />
  
- Here the issue is regarding Persistent Volume.
- See the flow below , what actually happens when we ask for volume as mentioned in the yml.
- **StorageClass** in Kubernetes tells the cluster what type of storage to create (fast, slow, SSD, etc.) and automatically provisions PersistentVolumes when a PVC asks for storage.  
<img width="874" height="70" alt="image" src="https://github.com/user-attachments/assets/32d839f3-3b5c-44c0-9af9-7efe6714979d" />
  
- Now ebs storage class maybe there on aws but here on minikube there's no storage class named **ebs**
<img width="872" height="210" alt="image" src="https://github.com/user-attachments/assets/f5cd293c-6bb3-4038-babf-a50ae8ada798" />
  
- On aws the flow maybe:
<img width="760" height="77" alt="image" src="https://github.com/user-attachments/assets/2a8f4136-528a-4f63-af3c-5293065b7c24" />
  
- Now here on Minikube the flow would be:
<img width="761" height="89" alt="image" src="https://github.com/user-attachments/assets/52941813-831e-4bd3-aa35-786ac4c171bd" />
  
- So go to the yaml file and change the storage class name to **standard**.
<img width="721" height="480" alt="image" src="https://github.com/user-attachments/assets/8106df0a-165f-46d4-b0c6-a4f9932bef57" />

<img width="578" height="76" alt="image" src="https://github.com/user-attachments/assets/a2d1f7c0-fd67-49a8-b08e-182e0ec664c6" />  

<img width="947" height="94" alt="image" src="https://github.com/user-attachments/assets/52ba4dc6-1701-4243-b745-626f90a44698" />  

<img width="644" height="205" alt="image" src="https://github.com/user-attachments/assets/d0e1b0dd-51d5-4ec9-a50b-9e537cfdd4dd" />  
  
- We also need to delete the exisitng PVC. So delete it and again deploy.

<img width="958" height="181" alt="image" src="https://github.com/user-attachments/assets/8efc8619-33b7-428a-95a8-57dcf542f4a0" />
  
<img width="906" height="407" alt="image" src="https://github.com/user-attachments/assets/211f969a-f209-42c5-9c12-528fe851c430" />

### CSI driver  

A CSI (Container Storage Interface) driver is a plugin that allows Kubernetes to connect to and provision storage from a storage system — whether it’s cloud storage (EBS, EFS) or external storage (NetApp, Dell EMC, etc.).  

Here we used the native storage services and providers like for aws , it's ebs, efs etc. But what if we want an external storage like Netapp. Now these external storage provides their own CSI ( Container storage Interface ) drivers which we need to install on our eks or any other cluster using helm or their official documentation. This driver will act as a storage provisioner or talk to the provisioner to create the external storage.The CSI driver lets Kubernetes talk to that storage system and dynamically create volumes using a StorageClass.External storage vendors provide their own StorageClasses along with their CSI driver.

<img width="728" height="56" alt="image" src="https://github.com/user-attachments/assets/dda8cdb7-9ffb-41f2-a633-5272829151c8" />




 




