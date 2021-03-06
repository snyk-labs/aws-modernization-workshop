+++
title = "Container Orchestration"
chapter = false
weight = 1
+++

## Container Orchestration

### Expected Outcome:
* 200 level usage of Amazon EKS

**Lab Requirements:**
* an Amazon Elastic Container Service for Kubernetes Cluster (EKS)

**Average Lab Time:** 20-30 minutes

### Introduction
In this module, we're going to deploy applications using http://aws.amazon.com/eks/[Amazon Elastic Container Service for Kubernetes (Amazon EKS)].

#### Getting Started with Amazon Elastic Container Service for Kubernetes (EKS)

Before we get started, here are some terms you need to understand in order to
deploy your application when creating your first Amazon EKS cluster.

* **Cluster:** A group of EC2 instances that are all running `kubelet` which
connects into the master control plane.
* **Deployment:** Configuration file that declares how a container should be
deployed including how many `replicas` what the `port` mapping should be and how
it is `labeled`.
* **Service:** Configuration file that declares the ingress into the container,
these can be used to create Load Balancers automatically.

##### Amazon EKS and Kubernetes Manifests
To use Amazon EKS or any Kubernetes cluster you must write a manifest or a
config file, these config files are used to declaritively document what is
needed to deploy an individual application. More information can be found
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/[here]

1. Switch to the tab where you have your Cloud9 environment opened.

2. In Cloud9 `cd` into the module directory.
```
cd ~/environment/aws-modernization-workshop/modules/container-orchestration-eks
```

3. Open the *petstore-eks-manifest.yaml* file by double clicking the filename
in the lefthand navigation in Cloud9.

4. The file has the following contents:

*.petstore-eks-manifest.yaml*
```
apiVersion: v1
kind: List
items:
- apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gp2
    annotations:
      "storageclass.kubernetes.io/is-default-class": "true"
  provisioner: kubernetes.io/aws-ebs
  parameters:
    type: gp2
  reclaimPolicy: Retain
  mountOptions:
    - debug
    
- apiVersion: v1
  kind: Namespace
  metadata:
    name: petstore

- kind: PersistentVolume
  apiVersion: v1
  metadata:
    name: postgres-pv-volume
    namespace: petstore
    labels:
      type: local
  spec:
    storageClassName: gp2
    capacity:
      storage: 20Gi
    accessModes:
      - ReadWriteOnce
    hostPath:
      path: "/mnt/data"

- apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: postgres-pv-claim
    namespace: petstore
  spec:
    storageClassName: gp2
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 20Gi

- apiVersion: v1
  kind: Service
  metadata:
    name: postgres
    namespace: petstore
  spec:
    ports:
    - port: 5432
    selector:
      app: postgres
    clusterIP: None

- apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: postgres
    namespace: petstore
  spec:
    selector:
      matchLabels:
        app: postgres
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          app: postgres
      spec:
        containers:
        - image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/petstore_postgres:latest
          name: postgres
          env:
          - name: POSTGRES_PASSWORD
            value: password
          - name: POSTGRES_DB
            value: petstore
          - name: POSTGRES_USER
            value: admin
          ports:
          - containerPort: 5432
            name: postgres
          volumeMounts:
          - name: postgres-persistent-storage
            mountPath: /var/lib/postgresql/data
            subPath: petstore
        volumes:
        - name: postgres-persistent-storage
          persistentVolumeClaim:
            claimName: postgres-pv-claim

- apiVersion: v1
  kind: Service
  metadata:
    name: frontend
    namespace: petstore
  spec:
    selector:
      app: frontend
    ports:
    - port: 80
      targetPort: http-server
      name: http
    - port: 9990
      targetPort: wildfly-cord
      name: wildfly-cord
    type: LoadBalancer

- apiVersion: apps/v1beta1
  kind: Deployment
  metadata:
    name: frontend
    namespace: petstore
    labels:
      app: frontend
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: frontend
    template:
      metadata:
        labels:
          app: frontend
      spec:
        initContainers:
        - name: init-frontend
          image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/petstore_postgres:latest
          command: ['sh', '-c',
                    'until pg_isready -h postgres.petstore.svc -p 5432;
                    do echo waiting for database; sleep 2; done;']
        containers:
        - name: frontend
          image: <YourAccountID>.dkr.ecr.us-west-2.amazonaws.com/petstore_frontend:latest
          resources:
            requests:
              memory: "512m"
              cpu: "512m"
          ports:
          - name: http-server
            containerPort: 8080
          - name: wildfly-cord
            containerPort: 9990
          env:
          - name: DB_URL
            value: "jdbc:postgresql://postgres.petstore.svc:5432/petstore?ApplicationName=applicationPetstore"
          - name: DB_HOST
            value: postgres.petstore.svc
          - name: DB_PORT
            value: "5432"
          - name: DB_NAME
            value: petstore
          - name: DB_USER
            value: admin
          - name: DB_PASS
            value: password
```

5. Replace the *<YourAccountID>* placeholders with your https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html[Account ID] and save the file.
```
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')
sed -i "s/<YourAccountID>/${ACCOUNT_ID}/" petstore-eks-manifest.yaml
```

6. Apply your manifest by running this command in your Cloud9 terminal:
```
kubectl apply -f petstore-eks-manifest.yaml
```

```
namespace/petstore created
persistentvolume/postgres-pv-volume configured
persistentvolumeclaim/postgres-pv-claim created
service/postgres created
deployment.apps/postgres created
service/frontend created
deployment.apps/frontend created
```

7. As you can see above this manifest created and configured several components
   in your Kubernetes cluster, we created a *namespace*, *persistentvolume*,
   *persistentvolumeclaim*, 2 *services*, and 2 *deployments*.

* **Namespace:** Namespaces are meant to be virtual clusters within a larger
pysical cluster.

* **PersistentValue:** Persistent Volume (PV) is a piece of storage that has been
provisioned by an administrator. _These are cluster wide resources._

* **PersistentVolumeClaim:** Persistent Volume Claim (PVC) is a request for storage
by a user.

* **Service:** Service is an abstraction which defines a logical set of Pods
and a policy by which to access them.

* **Deployment:** Deployment controller provides declarative updates for Pods and
ReplicaSets.

8. Now that the scheduler knows that you want to run this application it will
   find available *disk*, *cpu* and *memory* and will place the jobs. Let's
   watch as they get provisioned.
```
kubectl get pods --namespace petstore --watch
```

```
NAME                        READY     STATUS              RESTARTS   AGE
frontend-869db5db6b-ht4h8   0/1       Init:0/1            0          3m
frontend-869db5db6b-j5nfj   0/1       Init:0/1            0          3m
postgres-678864b7-vs5zj     0/1       ContainerCreating   5          3m
```

9. Once the *STATUS* changes to *Running* for all 3 of your containers we can
   then load the services and navigate to the exposed application (you will
   need to ctrl-c since its watching).
```
kubectl get services --namespace petstore -o wide
```

```
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)                                     AGE
frontend   LoadBalancer   10.100.20.251   ac7059d97a51611e88f630213e88d018-2093299179.us-west-2.elb.amazonaws.com   80:30327/TCP,443:32177/TCP,9990:30543/TCP   6m
postgres   ClusterIP      None            <none>                                                                    5432/TCP                                    6m
```

10. Here we can see that we're exposing the *frontend* using an ELB which is
   available at the *EXTERNAL-IP* field. Copy and paste this into a new browser
   tab.

11. Navigate to that URL with `/applicationPetstore` to see the running application

Now that we have our containers deployed to Amazon EKS we can continue with the workshop.
