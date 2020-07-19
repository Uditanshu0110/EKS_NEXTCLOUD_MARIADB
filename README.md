# EKS_NEXTCLOUD_MariaDB


## What is AWS EKS ?

Amazon EKS is a managed service that helps make it easier to run Kubernetes on AWS. Through EKS, organizations can run Kubernetes without installing and operating a Kubernetes control plane or worker nodes. Simply put, EKS is a managed **containers-as-a-service (CaaS)** that drastically simplifies Kubernetes deployment on AWS. EKS internally manages Master Node also.

In this article, we will know how we can launch the Kubernetes cluster using Amazon EKS service. For this, we have to first set up our system to work with Amazon EKS. There are some tasks that have to be done before we interact with the EKS service.


**AWS CLI should be installed in system.**

**System should have KUBECTL installed.**

**System should have EKSCTL installed.**

Now, let's understand what we will be doing throughout this development.

We will be Launching the Kubernetes cluster on the top of AWS using AWS EKS service. As EKS is a fully managed service this automatically creates worker nodes after launching clusters. After launching we will be launching our deployment into a launched cluster. Here we will be deploying NextCloud and MariaDB Database. After, launching we will use Prometheus and Grafana to monitor the launched cluster and resources associated with that.

**Now, let's move to our practical part.**

### STEP - 1:

Create an IAM user with Administrator Access Policy. This policy is a must to get involved and use the AWS EKS service. Make sure you have downloaded your credentials.

### STEP - 2:

Now, let's go to pre-installed AWS CLI. Here we will configure our credentials. Here we have to set our Access key, Secret key, and Region.

### STEP - 3:

Now let's create our Cluster File which we will use to launch a cluster in AWS.
Here we have to specify our cluster name, the region where we want to launch our cluster, instance type, and how much desire we have. In this example, the cluster name is - uditcluster, node group name is - node group-1, Instance type is - t2.micro and desired capacity - 5.

This file will launch Cluster named as uditcluster in the ap-south-1 region.

```apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: uditcluster
  region: ap-south-1
nodeGroups:
  - name: nodegroup-1
    instanceType: t2.micro
    desiredCapacity: 5
```
    
Generally, it takes 15-20 minutes to launch this whole setup.

### STEP - 4:

After launching the cluster we have to make some changes in kubeconfig files that belong to kubectl command.

```aws eks update-kubeconfig --name uditcluster```
This command will update kubeconfig file.

### STEP - 5:

Now, we will be creating our NameSpace. We use nameSpace because it helps to oraganize our code in better way. Also it reduces Name confilct error.

```kubectl cteate namespace myownns```
This command will create NameSpace myownns.

```kubectl config set-context --current --namespace=myownns```
This command will set our NameSpace default.

### STEP - 6:

Now It's time to deploy our deployment. Here we will be creating the YAML file for NextCloud and MariaDB DataBase.

In this file, we have specified Kind Service, Created PVC, and last but not the least Deployment part which is nothing but ReplicaSet. Here NextCloud by default uses the SQLite database but we have other options too like MariaDB, PostgreSQL, Mysql, and Redis. Here we will be using MariaDB but this same file will work fine with Mysql Database. This file automatically pulls images from the docker hub. Used Environment variables like MYSQL_ROOT_PASSWORD, MYSQL_PASSWORD, MYSQL_USER, MySQL_DATABASE. At last, we have mounted PVC to /var/lib/mysql because this is the main folder that stores data.

```apiVersion: v1
kind: Service
metadata:
  name: nextcloudsvc
  labels:
    app: nextcloudsvc
spec:
  ports:
  - port: 80
    nodePort: 30001
  selector:
    app: nextcloudsvc
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloudclaim
  labels:
    app: nextcloudsvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloudsvc
  labels:
    app: nextcloudsvc
spec:
  selector:
    matchLabels:
      app: nextcloudsvc
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nextcloudsvc
        tier: frontend
    spec:
      containers:
      - image: nextcloud:latest
        name: nextcloudsvc
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-certificate
              key: password
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadbuser-certificate
              key: password
        - name: MYSQL_USER
          value: uditanshu
        - name: MySQL_DATABASE
          value: mydatabase
        ports:
        - containerPort: 80
          name: nextcloudsvc
        volumeMounts:
        - name: nextcloud-storage
          mountPath: /var/www/html
      volumes:
      - name: nextcloud-storage
        persistentVolumeClaim:
          claimName: nextcloudclaim
```

### STEP - 6:

After creating the NEXTCLOUD deployment file we will be creating MARIADB file. It is the same file as above but a little bit modification has been done.

```apiVersion: v1
kind: Service
metadata:
  name: nextcloudmaria
  labels:
    app: nextcloud
spec:
  ports:
     - port: 3306
  selector:
    app: nextcloud
    tier: mariadb
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadbclaim
  labels: 
    app: nextcloud
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloudmaria
  labels:
    app: nextcloud
spec:
  selector:
   matchLabels:
     app: nextcloud
     tier: mariadb
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nextcloud
        tier: mariadb
    spec:
       containers:
       - image: mariadb:latest
         name: mariadb
         env:
         - name: MYSQL_ROOT_PASSWORD
           valueFrom:
             secretKeyRef:
              name: mariadb-certificate
              key: password
         - name: MYSQL_USER
           value: uditanshu
         - name: MYSQL_PASSWORD
           valueFrom:
             secretKeyRef:
               name: mariadbuser-certificate
               key: password
         - name: MYSQL_DATABASE
           value: mydatabase
         ports:
         - containerPort: 3306
           name: mysql
         volumeMounts:
         - name: mariadb-storage
           mountPath: /var/lib/mysql
       volumes:
       - name: mariadb-storage
         persistentVolumeClaim:
           claimName: mariadbclaim
```

### STEP - 7:

Now, we will create Kustomization file:

In this file, we will be specifying Secret and Resources. It is a standalone tool to customize Kubernetes objects. Must note the name of the file should be kustomization only.

```apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
secretGenerator:
- name: mariadb-certificate
  literals:
  - password=uditanshu
- name: mariadbuser-certificate
  literals:
  - password=redhat
resources:
  - maria.yml
  - nextcloud.yml
```

Command to run Kustomize file :
```kubectl create -k .```

This command will run kustomize file the current directory. So, no need to specify the name of the file. All the resources have been deployed.
Now as we have attached Service LoadBalancer in NextCloud Deployment. So, Aws launches An load balancer by using ELB (Elastic Load Balancing) service.
As this is a real LoadBalancer it takes some time for the complete setup. Now, we can connect to our deployment by using DNS name (This is a unique URL that can be given to any client who wants to connect to our deployment). 
Just create an admin account and you are good to go. We can also do this in our NextCloud Deployment file by using Environment Variables directly there. If we do that we have not to create the admin account.

Now we can also monitor our deployed resources using Prometheus and Grafana. ( This is optional ).

For this, we will require HELM. This should be installed in your system.

HELM is a package installer just like we use yum to install in RedHat os. But in Kubernetes world, we don't use the term Package. The package is known as CHART in the Kubernetes world. HELM can be downloaded from the Internet easily.
After downloading we have to Intiliaze and give permissions to the helm to install charts. For this, we will use some commands.

```helm init
helm repo update
```

```kubectl -n kube-system create serviceaccount tiller ```

```kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller ```

```helm repo add stable https://kubernetes-charts.storage.googleapis.com/```

Now, after this we will download Prometheus chart through Helm.

```kubectl create namespace prometheus```

This command will create NameSpace prometheus.

```helm install  stable/prometheus     --namespace prometheus     --set alertmanager.persistentVolume.storageClass="gp2"     --set server.persistentVolume.storageClass="gp2"```

```kubectl get svc -n prometheus```

Now, we will expose prometheus for client connection.

```kubectl -n prometheus  port-forward svc/flailing-buffalo-prometheus-server  8888:80```

This command will expose Prometheus and we can access that using IP Address - 12.0.0.1:8888. We can monitor our clusters and resources and metrics by using this.

For Graphical Representation or Visualization, we will be using Grafana.

```kubectl create namespace grafana```

This will Create NameSpace grafana.

```helm install stable/grafana  --namespace grafana     --set persistence.storageClassName="gp2" --set adminPassword='GrafanaAdm!n'    --set datasources."datasources\.yaml".apiVersion=1     --set datasources."datasources\.yaml".datasources[0].name=Prometheus   --set datasources."datasources\.yaml".datasources[0].type=prometheus    --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.prometheus.svc.cluster.local   --set datasources."datasources\.yaml".datasources[0].access=proxy     --set datasources."datasources\.yaml".datasources[0].isDefault=true  --set service.type=LoadBalancer```

This command will download grafana chart.

```kubectl get  secret  worn-bronco-grafana   --namespace  grafana  -o yaml```

After running these commands we can connect to grafana and use it.

Don't forget to clean and delete all the resources by running this command :

```eksctl delete cluster < Cluster Name >```


