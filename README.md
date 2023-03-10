
##  TASK Details

Deploy Mongo Database with a single master and two read replicas in the Kubernetes cluster of at least 3 worker nodes that are available in different availability zones.

Points to keep in mind while deploying the DB:
- All replicas of DB should be deployed in a separate worker node (For high availability).
- Autoscale the read replicas if needed.
- Data should be persistent.


## How to perform the above task

 
Create a StatefulSet for the MongoDB database with 3 replicas. This will ensure that each replica is deployed on a separate worker node, here I am using AWS cloud.


We will also need to create a persistent volume for each replica to ensure that the data is persistent, so here instead of Persistentvolumeclaim we will use VolumeClaimtemplate.

In Kubernetes, a VolumeClaimTemplate is a template that describes a desired PersistentVolumeClaim (PVC) configuration. PVCs are used to request a persistent storage resource from the cluster. 

The VolumeClaimTemplate is used to define the storage class, access mode, and storage size for the PVCs that will be created. When a pod is scheduled to run on a node, a PVC is automatically created based on the VolumeClaimTemplate configuration. This PVC is then bound to an available PersistentVolume (PV) with the appropriate storage size and access mode.














## These prerequisite should be there before implemented the task.

If you are using EKS cluster then, it should be deployed before implementing this task,
for the reference https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html

Below here you can see that I have created  node group with with 3 nodes that are available in different AZ's.

![eks](https://user-images.githubusercontent.com/127477253/224300862-c691c4f2-f739-45ff-889d-64b6c5b47989.png)





NOTE: When you deploy AWS EKS, so ensure that you have enabled AWS CSI Driver in add-ons
on the AWS console and attach the required IAM policies to it because we are going to use EBS volumes, for the reference https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html


## Create a headless service 
 
Example service.yaml 
```
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017 
```
Headless services are commonly used for StatefulSets, where each pod has a unique identity and state, and clients need to connect to each pod individually. By using a headless service, StatefulSets can perform domain name resolution (DNS) lookups to discover the network identity and IP addresses of the individual pods. 

This allows the client to connect directly to the pod they need to interact with, without relying on a load balancer to distribute traffic across the pods.

## Deploy the yaml file 
```
kubectl apply -f service.yaml
```



![svc](https://user-images.githubusercontent.com/127477253/224290822-bd4edd40-8d55-4c03-830a-6d28f03f951b.png)




## Create a statefulset file 
Example statefulset.yaml
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb 
  replicas: 2
  selector:
    matchLabels:
       app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
        selector: mongodb
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-east-1a
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                - us-east-1b 
    spec:
      containers: 
        - name: mongodb
          image: mongo:4.0.17
          ports:
            - containerPort: 27017
          readinessProbe:
            exec:
              command:
                - mongo
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - mongo
                - --eval
                - "db.adminCommand('ping')"
            initialDelaySeconds: 15
            periodSeconds: 10  
          resources:
            limits:
              cpu: "1"
              memory: "2Gi"
            requests:
              cpu: "0.5"
              memory: "1Gi"  
          volumeMounts:
            - name: pvc
              mountPath: /data/db           
  volumeClaimTemplates:
    - metadata:
        name: pvc
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp2
        resources:
          requests:
            storage: 1Gi
```
StatefulSets provide a way to manage stateful applications like MongoDB in a Kubernetes cluster, ensuring that data is persisted, and that pods have a stable and unique network identity, making it easier to scale and manage the application, for the referencee https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/            
## Deploy the yaml file 
```
kubectl apply -f statefulset.yaml
```

![po](https://user-images.githubusercontent.com/127477253/224291832-7a3459e8-5b67-4081-8fee-9bfeae29a99a.png)
## Create a HPA file for auto-scaling
Example hpa.yaml
```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: my-statefulset-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: mongodb
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
```
HPA stands for Horizontal Pod Autoscaler. It is a Kubernetes resource that automatically scales the number of replica pods in a Deployment, ReplicaSet, or StatefulSet based on the observed CPU utilization or custom metrics of the running pods.

The HPA controller monitors the CPU utilization or custom metrics of the replica pods and increases or decreases the number of replicas based on the set thresholds. This ensures that the application is always running at optimal capacity, with the right number of pods to handle the incoming traffic, for the reference https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
## Deploy the yaml file 
```
kubectl apply -f hpa.yaml
```

![hpa](https://user-images.githubusercontent.com/127477253/224292259-0745ce99-7a7d-428f-b155-3b7c4e8b36ea.png)
## Now you can also see on the AWS console our EBS volumes

![volumes](https://user-images.githubusercontent.com/127477253/224300380-8576edbb-b2bc-4cb0-9685-f215aabfd9ab.png)





So this was the complete description about the task.
