# k8s-vmware-airflow

## Create the kubernetes cluster on VMWare workstation

1. Download the minimal image for VMWare from https://www.linuxvmimages.com/images/centos-7/
2. Import into VMWare with the following settings:
    - Set CPUs 2/2 RAM to 4 GB
    - Turn on AMD RVI
    - Change network adapter from bridged to NAT
3. Start up the VM and login:
    - Set 
      ```localectl set-keymap hu```
    - Turn off SELINUX in /etc/selinux/config -> ```SELINUX=disabled```
    - Disable the firewall service with 
      - ```sudo systemctl disable firewalld```
    - Disable IPv6 (the apiserver defaulted listening on IPv6 making joining nodes not possible)
        - edit /etc/default/grub and add ```ipv6.disable=1``` to GRUB_CMDLINE_LINUX
        - ```grub2-mkconfig -o /boot/grub2/grub.cfg```

        - edit /etc/sysctl.conf
        ```
        net.ipv6.conf.all.disable_ipv6 = 1
        net.ipv6.conf.default.disable_ipv6 = 1
        ```
    - Enable bridge-nf-call in /usr/lib/sysctl.d/00-system.conf
        ```
        net.bridge.bridge-nf-call-ip6tables = 1
        net.bridge.bridge-nf-call-iptables = 1
        net.bridge.bridge-nf-call-arptables = 1
        ```
        ```sysctl -p /usr/lib/sysctl.d/00-system.conf```
    - Finally do ```yum update``` and restart the VM
4. Shutdown the VM and clone it 2 times
5. Follow the guide: https://medium.com/@hashithkarunarathne/create-a-kubernetes-cluster-from-the-scratch-on-centos-7-rhel-7-and-deploy-an-application-on-85a92997b565

6. If everything finished you should see:
```
  [centos@k8s-master ~]$ kubectl get nodes
  NAME                STATUS   ROLES    AGE   VERSION
  k8s-master          Ready    master   83m   v1.17.4
  k8s-worker-node-1   Ready    worker   18m   v1.17.4
  k8s-worker-node-2   Ready    worker   18m   v1.17.4
```

## Deploying MYSQL on the Kubernetes cluster

Using:
https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/
and
https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

All files can be found under the mysql directory in this repository.

1. Create the Persistent Volume Claim (mysql-pv.yaml)
2. Create the mysql secret (kustomization.yaml)
3. Create the mysql deployment (mysql-deployment.yaml)
4. Prepare MYSQL for Airflow
    Using the tutorial @ https://opensource.com/article/19/6/introduction-kubernetes-secrets-and-configmaps
    Create the configmap and apply to mysql
    kubectl create configmap mysql-cnf --from-file=./my-config.cnf
    


Logged in to the master node and for example in /home/centos/mysql/
the command ```kubectl apply -k .``` should create our deployment.

```
[centos@k8s-master mysql]$ kubectl get deployment mysql
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
mysql   1/1     1            1           18h
```

```
[centos@k8s-master mysql]$ kubectl describe pod mysql-db7b67f8b-tkzrl
Name:         mysql-db7b67f8b-tkzrl
Namespace:    default
Priority:     0
Node:         k8s-worker-node-2/192.168.192.130
Start Time:   Sat, 14 Mar 2020 13:39:03 -0400
Labels:       app=mysql
              pod-template-hash=db7b67f8b
Annotations:  <none>
Status:       Running

```

Check to see if the mysql configmap got applied properly:
```
kubectl exec -ti $(kubectl get po -l app=mysql -o jsonpath="{.items[0].metadata.name}") -- mysql -u root -p -e "SHOW VARIABLES LIKE 'explicit_defaults_for_timestamp'"
```

Expected is ON:
```
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| explicit_defaults_for_timestamp | ON    |
+---------------------------------+-------+
```

Login to the mysql database and issue the following statement:
```
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pmysqltest123
```
```
GRANT ALL PRIVILEGES ON *.* TO 'airflow'@'%' IDENTIFIED BY 'airflow' WITH GRANT OPTION;
flush privileges;
```

## Setting up Airflow

1. Register on Docker hub and set up a repository: https://hub.docker.com/repository/docker/aungvari/test
2. Using the guide https://github.com/stwind/airflow-on-kubernetes build the docker image.
   - Download Dockerfile from this repository and use the command ```sudo docker build -t airflow .```
   - Push it to our repository with ```sudo docker push aungvari/test:airflow```
3. Following the guide initialize the airflow database:

```
kubectl run airflow-initdb \
    --restart=Never -ti --rm --image-pull-policy=IfNotPresent --generator=run-pod/v1 \
    --image=aungvari:test/airflow \
    --env AIRFLOW__CORE__LOAD_EXAMPLES=False \
    --env AIRFLOW__CORE__SQL_ALCHEMY_CONN=mysql://airflow:airflow@mysql.default/airflow \
    --command -- airflow initdb
```

4. You should see the tables getting created in MYSQL in the airflow database
    - Login to the mysql pod
    - ```use airflow;```
    - ```show tables;```
    
5. Run Airflow
    
```
kubectl run airflow -ti --rm --restart=Never --image=aungvari/test:airflow --overrides='
{
  "spec": {
    "containers":[{
      "name": "webserver",
      "image": "aungvari/test:airflow",
      "imagePullPolicy":"IfNotPresent",
      "command": ["airflow","webserver"],
      "stdin": true,
      "tty": true,
      "env": [
        {"name":"AIRFLOW__CORE__LOAD_EXAMPLES","value":"False"},
        {"name":"AIRFLOW__CORE__SQL_ALCHEMY_CONN","value":"mysql://airflow:airflow@mysql.default/airflow"},
        {"name":"AIRFLOW__CORE__EXECUTOR","value":"KubernetesExecutor"},
        {"name":"AIRFLOW__KUBERNETES__WORKER_CONTAINER_REPOSITORY","value":"aungvari/test:airflow"},
        {"name":"AIRFLOW__KUBERNETES__WORKER_CONTAINER_TAG","value":"latest"},
        {"name":"AIRFLOW__KUBERNETES__DAGS_VOLUME_HOST","value":"'$PWD/dags'"}
      ],
      "volumeMounts": [{"mountPath": "/var/lib/airflow/dags","name": "store"}]
    },{
      "name": "scheduler",
      "image": "aungvari/test:airflow",
      "imagePullPolicy":"IfNotPresent",
      "command": ["airflow","scheduler"],
      "stdin": true,
      "tty": true,
      "env": [
        {"name":"AIRFLOW__CORE__LOAD_EXAMPLES","value":"False"},
        {"name":"AIRFLOW__CORE__SQL_ALCHEMY_CONN","value":"mysql://airflow:airflow@mysql.default/airflow"},
        {"name":"AIRFLOW__CORE__EXECUTOR","value":"KubernetesExecutor"},
        {"name":"AIRFLOW__KUBERNETES__WORKER_CONTAINER_REPOSITORY","value":"aungvari/test:airflow"},
        {"name":"AIRFLOW__KUBERNETES__WORKER_CONTAINER_TAG","value":"latest"},
        {"name":"AIRFLOW__KUBERNETES__DAGS_VOLUME_HOST","value":"'$PWD/dags'"},
        {"name":"AIRFLOW__KUBERNETES__KUBE_CLIENT_REQUEST_ARGS","value":""}
      ],
      "volumeMounts": [{"mountPath": "/var/lib/airflow/dags","name": "store"}]
    }],
    "volumes": [{"name":"store","hostPath":{"path":"'$PWD/dags'"}}]
  }
}'
```
6. Check the status of the pods:

```
[centos@k8s-master ~]$ kubectl describe pod airflow
Name:         airflow
Namespace:    default
Priority:     0
Node:         k8s-worker-node-1/192.168.192.129
Start Time:   Sun, 15 Mar 2020 04:25:37 -0400
Labels:       run=airflow
```

```
Events:
  Type    Reason     Age        From                        Message
  ----    ------     ----       ----                        -------
  Normal  Scheduled  <unknown>  default-scheduler           Successfully assigned default/airflow to k8s-worker-node-1
  Normal  Pulled     21s        kubelet, k8s-worker-node-1  Container image "aungvari/test:airflow" already present on machine
  Normal  Created    21s        kubelet, k8s-worker-node-1  Created container webserver
  Normal  Started    21s        kubelet, k8s-worker-node-1  Started container webserver
  Normal  Pulled     21s        kubelet, k8s-worker-node-1  Container image "aungvari/test:airflow" already present on machine
  Normal  Created    21s        kubelet, k8s-worker-node-1  Created container scheduler
  Normal  Started    21s        kubelet, k8s-worker-node-1  Started container scheduler
```

7. We can now connect to the airflow webserver if we forward the port 8080:

``` [centos@k8s-master ~]$ kubectl port-forward airflow 8080 ```

8. After setting up SSH tunneling to the PC we can access ```http://localhost:8080/admin/queryview/``` where it is possible to run ad-hoc queries on the database.
It is possible to upload the content of a .csv from the filesystem into the database via SQL. NOTE: we need to remove the password for user "root" in MYSQL for this to work temporarily.
https://dev.mysql.com/doc/refman/5.6/en/load-data.html

First we need to create the table and the columns:
```
CREATE TABLE stores (
    storenr INT NOT NULL PRIMARY KEY,
    close_date VARCHAR(255) NOT NULL,
    address VARCHAR(255) NOT NULL,
	warehousenr INT NOT NULL
);
```
This is getting created properly:

![kép](https://user-images.githubusercontent.com/12872375/76699527-80d4e500-66ae-11ea-8f4b-786fa5e10c69.png)

Then copy the stores.csv to the container with: 
```
[centos@k8s-master ~]$ kubectl cp stores.csv airflow:/tmp/
Defaulting container name to webserver.
```

Trying to upload the data into MYSQL:

```
LOAD DATA LOCAL INFILE '/tmp/stores.csv'
INTO TABLE stores
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(storenr,close_date,address,warehousenr)
```

Unfortunately Airflow errors here and no data gets uploaded:

![kép](https://user-images.githubusercontent.com/12872375/76699670-1624a900-66b0-11ea-87fc-e2cf0710f99b.png)


## Extending the project

1. First and foremost there are no security considerations involved at any point in the above: 
    - Using the kubernetes secret store is not recommended as passwords are only base64 encoded
    - Passwords for airflow and root user in mysql/linux are defaults and/or unsafe
    - No user/login is setup for airflow WEB UI (https://airflow.apache.org/docs/stable/security.html)
    - SSL and HTTPS needs to be setup for Airflow
    - Firewalls on the VMs are shutdown instead of opening only the required ports for the Kubernetes services (https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
    - Kubernetes RBAC configurations are all defaults
    
2. Provisioning the infrastructure
    - Setting up the k8s cluster be done in a public cloud with terraform and the kubernetes provider or using an already existing solution like https://coreos.com/tectonic/docs/latest/install/azure/azure-terraform.html
    - Terraform also support vSphere API for creating VMs (not workstation though)
    - Configurations on the OS level like IPv6 kernel parameters, swap settings, hostnames, kubernetes and docker repositories could be done with packer and this image would be used by terraform
    - Installing and starting the services like docker and kubectl/kubeadm/kubelet should be done with a configuration management tool (chef/ansible/puppet/salt)
    - Joining the nodes to the cluster via token, setting up the flannel network and labelling the nodes should be also done with configuration management
    - Existing standard Helm charts could be used for MYSQL/Airflow/Redis/RabbitMQ (https://github.com/helm/charts/tree/master/stable/airflow)

3. Airflow
    - For a production landscape the CeleryExecutor should be used which requires additional services like RabbitMQ and Redis
    - Creating a workflow and schedule (DAG?) to import the .csv files with a python program
    - The pod needs to be put into a yaml file instead of being run with ```kubectl run airflow```
    
4. Monitoring and logging
    - Log files from Airflow and MYSQL should be stored persistently in an external application like logstash
    - Elasticsearch is capable of indexing metrics and logs fed to it from logstash
    - Grafana or Kibana can be used to visualize data
    - Alerting when kubernetes services go down (heartbeat)
    
## Additional comments
1. Airflow has a pretty steep learning curve :)
2. YAML is unforgiving
3. It is very difficult to find any tutorials which go beyond the basics or are not already the complete solution
4. Python knowledge is absolutely necessary going forward
