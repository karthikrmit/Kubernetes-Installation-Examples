## Install Kubernetes v1.10.11 with Flannel

The below installation is with single master and single worker node

### On Master Node

1. Setup passphraseless ssh between master and worker nodes(optional)

2. Create a kubernetes.repo file under `/etc/yum.repos.d/kubernetes.repo`

```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

3. Disable selinux by running the below command 

```
setenforce 0
```

4. Add the Docker public key for CS Docker Engine packages

```
rpm --import "https://sks-keyservers.net/pks/lookup?op=get&search=0xee6d536cf7dc86e2d7d56f59a178ac6c6238f52e"
```

5. Install yum-utils if needed (optional in most cases)

```
yum install -y yum-utils
```

6 Add repo for docker 1.13 
```
yum-config-manager --add-repo https://packages.docker.com/1.13/yum/repo/main/centos/7
```

7. Install docker 1.13 and Kubernetes 1.10.11
```
yum install -y docker-engine-1.13.1 docker-engine-selinux-1.13.1 kubelet-1.10.11 kubeadm-1.10.11 kubectl-1.10.11 kubernetes-cni-0.6.0
```

8. Set the cgroup driver to systemd (Depends on the host machine)

```
Replace the line in /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd

by

ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
```

9. Enable and start docker container runtime
```
systemctl daemon-reload && systemctl enable docker && systemctl start docker
```

10. Start the kubelet
```
systemctl enable kubelet && systemctl start kubelet
```

11. Check the status of kubelet. It will be failing at the moment. That's fine !
```     
[root@master-node ~]# systemctl status kubelet -l
\u25cf kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           \u2514\u250010-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Thu 2018-02-07 16:40:33 PST; 3s ago
     Docs: https://kubernetes.io/docs/
  Process: 21797 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 21797 (code=exited, status=255)

Feb 07 16:40:33 master-node.my-cluster.com systemd[1]: kubelet.service: main process exited, code=exited, status=255/n/a
Feb 07 16:40:33 master-node.my-cluster.com systemd[1]: Unit kubelet.service entered failed state.
Feb 07 16:40:33 master-node.my-cluster.com systemd[1]: kubelet.service failed.
```

12. To avoid issues with traffic being routed incorrectly due to iptables being bypassed(pass bridged IPv4 traffic to iptables’ chains)
```
sysctl net.bridge.bridge-nf-call-iptables=1
```

13. Change the cluster dns to 192.168.0.10 if needed in `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` to avoid any conflict with your underlay IPs if they are in 10.x.x.x range

14. Disable swap by `swapoff -a`

15. Install Kubernetes using kubeadm init
```
kubeadm init --kubernetes-version v1.10.11  --service-cidr 192.168.0.0/16 --node-name master-node.my-cluster.com --cert-dir /etc/kubernetes/pki --token-ttl 0 
```

16. The successful output will look like below
```
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.x.x.x:6443 --token btwb5x.vq0tej3nbg7bjzsq --discovery-token-ca-cert-hash sha256:6786a7cffa591b290aaae55de7aef709d1be17b766829efddd937458a465fa6c
```

17. If you check kubelet now, it should be successfully running.
```
 [root@master-node ~]# service kubelet status -l
Redirecting to /bin/systemctl status  -l kubelet.service
\u25cf kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           \u2514\u250010-kubeadm.conf
   Active: active (running) since Thu 2018-02-07 16:46:51 PST; 10min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 22564 (kubelet)
    Tasks: 14
   Memory: 44.7M
   CGroup: /system.slice/kubelet.service
```
   

### Install Kubernetes on the worker nodes and join them to the master to form a cluster

1. Create a kubernetes.repo file under `/etc/yum.repos.d/kubernetes.repo`
```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

2. Disable selinux by running the below command 
```
setenforce 0
```

3. Add the Docker public key for CS Docker Engine packages
```
rpm --import "https://sks-keyservers.net/pks/lookup?op=get&search=0xee6d536cf7dc86e2d7d56f59a178ac6c6238f52e"
```

4. Add repo for docker 1.13 
```
yum-config-manager --add-repo https://packages.docker.com/1.13/yum/repo/main/centos/7
```

5. Install docker 1.13 and Kubernetes 1.10.11
```
yum install -y docker-engine-1.13.1 docker-engine-selinux-1.13.1 kubelet-1.10.11 kubeadm-1.10.11 kubectl-1.10.11 kubernetes-cni-0.6.0
```

6. Set the cgroup driver to systemd(Depends on the host machine)
```
Replace the line in /usr/lib/systemd/system/docker.service

ExecStart=/usr/bin/dockerd

by

ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
```

7. Enable and start docker container runtime
```
systemctl daemon-reload && systemctl enable docker && systemctl start docker
```

8. To avoid issues with traffic being routed incorrectly due to iptables being bypassed(pass bridged IPv4 traffic to iptables’ chains).
```
sysctl net.bridge.bridge-nf-call-iptables=1
```

9. Change the cluster dns to 192.168.0.10 if needed in `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` to avoid any conflict with your underlay IPs if they are in 10.x.x.x range

10. Enable and start the kubelet
```
systemctl enable kubelet && systemctl start kubelet
```

11. Check the status of kubelet. It will be failing at the moment. That's fine !
```     
 [root@worker-node1 ~]# systemctl status kubelet
\u25cf kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           \u2514\u250010-kubeadm.conf
   Active: activating (auto-restart) (Result: exit-code) since Thu 2018-02-07 17:00:14 PST; 4s ago
     Docs: https://kubernetes.io/docs/
  Process: 12098 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=255)
 Main PID: 12098 (code=exited, status=255)

Feb 07 17:00:14 worker-node1.my-cluster.com systemd[1]: Unit kubelet.service entered failed state.
Feb 07 17:00:14 worker-node1.my-cluster.com systemd[1]: kubelet.service failed.
```

12. Disable swap by `swapoff -a`

13. Run kubeadm join with `--ignore-preflight-errors=cri` (Only for Kubernetes 1.10.11 as this seems to be an issue in my view)
```
kubeadm join 10.x.x.x:6443 --token btwb5x.vq0tej3nbg7bjzsq --discovery-token-ca-cert-hash sha256:6786a7cffa591b290aaae55de7aef709d1be17b766829efddd937458a465fa6c --ignore-preflight-errors=cri
[preflight] Running pre-flight checks.
```

14. A successful join will look like below
```
This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

15. Verify the cluster installation on master node

Run below command to see all the nodes that are part of the cluster
```
[root@master-node ~]# kubectl get nodes
NAME                                     STATUS     ROLES     AGE       VERSION
worker-node1.my-cluster.com    NotReady   <none>    1m        v1.10.11
master-node.my-cluster.com   NotReady   master    17m       v1.10.11
```

**Note:**

The nodes will be NotReady state until we deploy a Network Plugin (Flannel in this case).


## Deploy Flannel Network Plugin

**Note:**

a. For flannel to work correctly, you must pass `--pod-network-cidr=10.244.0.0/16` to kubeadm init.

b. If you want to use a different pod-network-cidr, for example 70.70.0.0/16, you must pass `--pod-network-cidr=70.70.0.0/16` to kubeadm init

1. Download Flannel Network yaml 
```
wget https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

2. Change the pod network cidr in kube-flannel.yml if you used a pod-network-cidr other than 10.244.0.0/16
```
  net-conf.json: |
    {
      "Network": "70.70.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
```

3. Apply flannel network plugin
```
kubectl apply -f kube-flannel.yml
```

4. A successful flannel deployment will look like below
```
[root@master-node ~]# kubectl get pods --all-namespaces 
NAMESPACE     NAME                                                 READY     STATUS    RESTARTS   AGE
kube-system   etcd-master-node.my-cluster.com                      1/1       Running   0          8m
kube-system   kube-apiserver-master-node.my-cluster.com            1/1       Running   0          8m
kube-system   kube-controller-manager-master-node.my-cluster.com   1/1       Running   0          8m
kube-system   kube-dns-86f4d74b45-sktn6                            3/3       Running   0          9m
kube-system   kube-flannel-ds-amd64-5h76c                          1/1       Running   0          8m
kube-system   kube-flannel-ds-amd64-rxz9w                          1/1       Running   0          8m
kube-system   kube-proxy-2cwk4                                     1/1       Running   0          8m
kube-system   kube-proxy-hkm7f                                     1/1       Running   0          9m
kube-system   kube-scheduler-master-node.my-cluster.com            1/1       Running   0          8m
```

## Deploy a PHP Guestbook application

1. Create a guestbook namespace
```
[root@master-node ~]# kubectl create namespace guestbook
namespace "guestbook" created
```

2. Create redis master deployment
```
[root@master-node ~]# kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-deployment.yaml -n guestbook
deployment.apps "redis-master" created
```

3. Verify redis master pod is Running
```
[root@master-node ~]# kubectl get all -n guestbook -o wide
NAME                                READY     STATUS    RESTARTS   AGE       IP          NODE
pod/redis-master-55db5f7567-znq54   1/1       Running   0          1m        70.70.1.4   worker-node1.my-cluster.com

NAME                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                 SELECTOR
deployment.apps/redis-master   1         1         1            1           1m        master       k8s.gcr.io/redis:e2e   app=redis,role=master,tier=backend

NAME                                      DESIRED   CURRENT   READY     AGE       CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/redis-master-55db5f7567   1         1         1         1m        master       k8s.gcr.io/redis:e2e   app=redis,pod-template-hash=1186193123,role=master,tier=backend
```

4. Check that the redis master is running
```
[root@master-node ~]# kubectl logs -f pod/redis-master-55db5f7567-znq54 -n guestbook
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 2.8.19 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

[1] 08 Feb 18:23:15.880 # Server started, Redis version 2.8.19
[1] 08 Feb 18:23:15.881 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
[1] 08 Feb 18:23:15.881 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
[1] 08 Feb 18:23:15.881 * The server is now ready to accept connections on port 6379
```

5. Create a redis-master service
```
[root@master-node ~]# kubectl apply -f https://k8s.io/examples/application/guestbook/redis-master-service.yaml -n guestbook
service "redis-master" created

[root@master-node ~]# kubectl get svc -n guestbook
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
redis-master   ClusterIP   192.168.106.61   <none>        6379/TCP   11s
```

6. Create a redis-slave deployment with redis slave pods
```
[root@master-node ~]# kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-deployment.yaml -n guestbook
deployment.apps "redis-slave" created
```

7. Create a redis-slave service
```
[root@master-node ~]# kubectl apply -f https://k8s.io/examples/application/guestbook/redis-slave-service.yaml -n guestbook
service "redis-slave" created

[root@master-node ~]# kubectl get pods -n guestbook -o wide
NAME                            READY     STATUS    RESTARTS   AGE       IP          NODE
redis-master-55db5f7567-znq54   1/1       Running   0          3m        70.70.1.4   worker-node1.my-cluster.com
redis-slave-584c66c5b5-nbmhc    1/1       Running   0          39s       70.70.1.6   worker-node1.my-cluster.com
redis-slave-584c66c5b5-nq9vg    1/1       Running   0          39s       70.70.1.5   worker-node1.my-cluster.com

[root@master-node ~]# kubectl get svc -n guestbook
NAME           TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)    AGE
redis-master   ClusterIP   192.168.106.61    <none>        6379/TCP   1m
redis-slave    ClusterIP   192.168.203.216   <none>        6379/TCP   9s
```

8. Create the frontend deployment/pods to access the redis master/slave
```
[root@master-node ~]# kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-deployment.yaml -n guestbook
deployment.apps "frontend" created

[root@master-node ~]# kubectl get pods -l app=guestbook -l tier=frontend -n guestbook
NAME                        READY     STATUS    RESTARTS   AGE
frontend-5c548f4769-f8czr   1/1       Running   0          1m
frontend-5c548f4769-k96dp   1/1       Running   0          1m
frontend-5c548f4769-ld64j   1/1       Running   0          1m
```

9. Create the frontend service(NodePort service to access the webpage from outside the cluster)
```
[root@master-node ~]#  kubectl apply -f https://k8s.io/examples/application/guestbook/frontend-service.yaml -n guestbook
service "frontend" created

[root@master-node ~]# kubectl get svc -n guestbook
NAME           TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)        AGE
frontend       NodePort    192.168.75.230    <none>        80:31273/TCP   7s
redis-master   ClusterIP   192.168.106.61    <none>        6379/TCP       3m
redis-slave    ClusterIP   192.168.203.216   <none>        6379/TCP       2m
```

10. Verify the NodePort service from the master/worker node
```
[root@master-node ~]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.31.32.68  netmask 255.255.224.0  broadcast 10.31.63.255
        inet6 fe80::f8ac:a6ff:fe02:e00  prefixlen 64  scopeid 0x20<link>
        ether fa:ac:a6:02:0e:00  txqueuelen 1000  (Ethernet)
        RX packets 8541185  bytes 6841633700 (6.3 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 372292  bytes 221228946 (210.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@master-node ~]# curl -k 10.31.32.68:31273
<html ng-app="redis">
  <head>
    <title>Guestbook</title>
    <link rel="stylesheet" href="//netdna.bootstrapcdn.com/bootstrap/3.1.1/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.2.12/angular.min.js"></script>
    <script src="controllers.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/angular-ui-bootstrap/0.13.0/ui-bootstrap-tpls.js"></script>
  </head>
  <body ng-controller="RedisCtrl">
    <div style="width: 50%; margin-left: 20px">
      <h2>Guestbook</h2>
    <form>
    <fieldset>
    <input ng-model="msg" placeholder="Messages" class="form-control" type="text" name="input"><br>
    <button type="button" class="btn btn-primary" ng-click="controller.onRedis()">Submit</button>
    </fieldset>
    </form>
    <div>
      <div ng-repeat="msg in messages track by $index">
        {{msg}}
      </div>
    </div>
    </div>
  </body>
</html>
```

## Reference: 

https://kubernetes.io/docs/setup/independent/install-kubeadm/

https://kubernetes.io/docs/tutorials/stateless-application/guestbook/
