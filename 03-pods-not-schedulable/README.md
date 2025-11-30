In Kubernetes, the scheduler is responsible for assigning pods to nodes in the cluster based on various criteria. Sometimes, you might encounter situations where pods are not being scheduled as expected. This can happen due to factors such as node constraints, pod requirements, or cluster configurations.

## 1. Node Selector

Node Selector is a simple way to constrain pods to nodes with specific labels. It allows you to specify a set of key-value pairs that must match the node's labels for a pod to be scheduled on that node.  
Usage: Include a nodeSelector field in the pod's YAML definition to specify the required labels.

```
spec:
    containers:
    - name: my-app
    image: my-image
    nodeSelector:
    node-name: arm-worker
```

Now if the none of the nodes are having this label then the pods will not be scheduled to any of the nodes and it will cause **failed scheduling**. Check the pods status using `kubectl describe pod <pod-name` and the results below:
<img width="975" height="469" alt="image" src="https://github.com/user-attachments/assets/dccfe7bd-916d-4604-a16b-ac8b86e8eec2" />

give label to one of the node or more than 1 . 
- `kubectl get nodes`
- `kubectl edit node <node-name>`
<img width="802" height="365" alt="image" src="https://github.com/user-attachments/assets/30c9153f-030f-46f3-a7d3-e0c95bf34caa" />

Now u will see that the pods are assigned to that labeled node.
  
## 2. Node Affinity
  
Node Affinity is a more expressive way to specify rules about the placement of pods relative to nodes' labels. It allows you to specify rules that apply only if certain conditions are met.  
Usage: Define nodeAffinity rules in the pod's YAML definition, specifying required and preferred node selectors.

```
spec:
    containers:
    - name: my-app
    image: my-image
    affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
            - key: node-name
            operator: In
            values:
            - arm-worker
```

Node Affinity helps you control Pod placement when you want pods to run on a specific set of nodes  
**🔷 Types of Node Affinity**  
**1️⃣ RequiredDuringSchedulingIgnoredDuringExecution**
- Hard rule
  - Scheduler must place the Pod on a matching node
  - If no matching node exists → Pod stays pending
- Example:
  - "Only run on nodes labeled env=prod."  
**2️⃣ PreferredDuringSchedulingIgnoredDuringExecution**
- Soft rule
  - Scheduler tries to place Pod on preferred nodes but can fall back to other nodes if the no node with that label is present
  - Pod will still run even if no nodes match
- Example:
  - "Prefer to run on env=prod nodes, but run anywhere if not available."  
👉 In short:  
- Node Selector = simple + strict
- Node Affinity = flexible + supports soft & hard rules
  
## 3. Taints
  
Taints are applied to nodes to repel certain pods. They allow nodes to refuse pods unless the pods have a matching toleration.  
- Taint = a rule on a node that prevents unwanted pods from scheduling on it.
- Pods can run on that tainted node only if they have a matching toleration.  
Usage: Use kubectl taint command to apply taints to nodes. Include tolerations field in the pod's YAML definition to tolerate specific taints.
Use-case: There are 2 nodes in a cluster and u want to upgrade them so u will do one by one , u will shift all the pods to the other nodes and then apply Noschedule rule for that particular pod and bring it down , upgrade it and then now ur node is ready and similarily u can do for other nodes too.    
```
kubectl taint nodes node1 node-name=arm-worker:NoSchedule
```
This means:  
> *Do NOT schedule pods on this node unless they tolerate this taint*

❗ Result:
> - Pods without toleration → will NOT be scheduled on this node.
> - Pods with matching toleration → allowed to run on this node.
> - So u dont need to do anything in the yaml of pods , the **taint** command just restricted all the pods formation on it unless and until u add **tolerations** inisde the yaml of that particular deployment or pod.
> - That taint says: “Do NOT schedule any pod here unless it has a matching toleration.”  

**🔹 Taint Effects**
| Effect               | Meaning                                        |
| -------------------- | ---------------------------------------------- |
| **NoSchedule**       | Pods without toleration **won't be scheduled**. Scheduler will not place new Pods on this node unless they have matching toleration. |
| **PreferNoSchedule** | Try not to schedule, but not strict            |
| **NoExecute**        | Evicts running pods without toleration         |

4. Tolerations

- Tolerations are applied to pods and allow them to schedule onto nodes with matching taints. They override the effect of taints.
- A toleration is a setting you add in a pod's YAML that allows the pod to run on a node that has a taint.  
Usage: Include tolerations field in the pod's YAML definition to specify which taints the pod tolerates.

```
spec:
  containers:
  - name: my-app
    image: my-image
  tolerations:
  - key: node-name
    operator: Equal
    value: arm-worker
    effect: NoSchedule
```
This means:
> This pod matches and tolerates the taint node-name=arm-worker:NoSchedule.
  
💡 Meaning:
> - "This pod is allowed to run on nodes that have the taint node-name=arm-worker:NoSchedule."
> - It means now this pod can be scheduled to the **node 1** as the node 1 has that label with Noschedule taint effect and except this deployment pods , other pods are not allowed to schedule on that node as they dont have tolerations inside their yaml.
