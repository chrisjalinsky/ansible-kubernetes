Kubernetes Cluster:
===================

Dependencies:
* Ansible 2.0
* Vagrant 1.8.1
* Virtualbox 3.4.43

You probably only need to have Ansible installed to get this environment up and running. Esp. if you are deploying to bare-metal or another hypervisor. You'll have to create a simple inventory file. Look at ansible/hosts.yaml or ansible/static_inventory to see what groups the playbooks use.

Create Environment with Vagrant
===============================

I use vagrant-hostsupdater plugin to write to my Macbook's /etc/hosts file for DNS resolution. This is so I can resolve hostnames in my browser. There is a playbook that installs DNS into the private network, as well as updates resolv.conf to utilize the newly created DNS server. You can skip these playbooks if your environment contains DNS resolution.
```
vagrant up
```

Once instances are finished building, you can run all playbooks with the following command. The stderr/stdout of this script is appended to a file named ./install.out. The playbooks and shell script use a dynamic inventory source ```-i inventory.py``` which is just a placeholder. The dynamic inventory script simply parses the hosts.yaml file. Included is a static inventory file as well, ```ansible/static_inventory```.

```
ansible/run_playbooks.sh
```

You can run the following playbooks by hand if you wish.

Ansible playbooks:
==================

These playbooks use an ansible.cfg file (bypass known_hosts conflicts and host key checking--use appropriately):

Test whether you can reach the hosts:
-------------------------------------
```
ansible all -i inventory.py -m ping
```

Add DNS to the cluster (if you choose a different IP range for your hosts, be sure to include the correct zones for DNS usage):
------------------
```
ansible-playbook provision_core_servers.yaml -i inventory.py
```

Update nodes to use DNS server:
-----------------
```
ansible-playbook update_resolv.yaml -i inventory.py
```

Add ssh key for root:
--------------
```
ansible-playbook add_user_for_kubernetes.yaml -i inventory.py
```

Deploy Docker:
--------------
```
ansible-playbook provision_docker_servers.yaml -i inventory.py
```

Deploy Docker registry:
-----------------------
This starts an upstart script which calls ```./docker-compose up``` in the ```/docker-registry``` folder
```
ansible-playbook provision_docker_registry_servers.yaml -i inventory.py
```

Copy Docker Registry cert from kubernetes master to docker engines:
-------------------
```
ansible-playbook delegate_copy_ssl_cert.yaml -i inventory.py -vv
```

Build Kubernetes cluster for ubuntu:
----------------------------
```
ansible-playbook provision_kubernetes.yml -i inventory.py
```

Docker build app to test deployment:
------------------------
```
ansible-playbook docker_build_app.yml -i inventory.py
```

Then go to kmas1.lan:/tmp/docker_app/:
---------------------------
```
docker build -t kmas1.lan/hello-node:v1 .
```

Install Monitoring and Heapster:
-----------------
```
git clone https://github.com/kubernetes/heapster.git
```

The above steps are handled in the run_playbooks.sh script.

---

After the build, test it out by running natively in docker and exposing port 8090:
```
docker run -d -p 8090:8080 kmas1.lan/hello-node:v1
```

Kill the container:
```
docker kill kmas1.lan/hello-node:v1 
```

Push to the new docker registry so that we can use it in Kubernetes:
```
docker push kmas1.lan/hello-node:v1
```

Building Kubernetes pods and exposing them. Using the secure nginx example (Cant create the secrets yet):

Install go via this link:
```
http://www.hostingadvice.com/how-to/install-golang-on-ubuntu/

```
To create a django
kubectl create -f ./django-redis-pod.yaml
vi django-redis-pod.yaml
```
---

apiVersion: v1
kind: Pod
metadata:
  name: redis-django
  labels:
    app: web
spec:
  containers:
    - name: key-value-store
      image: redis
      ports:
        - containerPort: 6379
    - name: frontend
      image: django
      ports:
        - containerPort: 8000
```

Navigate to examples https example:
```
cd /opt/kubernetes/examples/https-nginx/

```

Using the Docker Registry
==================

Here's a simple use case:

```
sudo docker login https://kmas1.lan
```

There are 2 test users defined, so sign in with the credentials defined in vars/makevault.yml. This should be converted to an ansible-vault, but for testing purposes, it's left open.

At the login prompt:

```
username: testuser
password: pass
email: me@example.com
```

Once logged in, use the registry (if correctly configured, root user can run docker commands below, so no sudo is necessary):

```
docker pull ubuntu

docker tag ubuntu kmas1.lan/ubuntu

docker push kmas1.lan/ubuntu

docker pull kmas1.lan/ubuntu

```


Docker Registry Systemd Xenial 64 (not applicable to trusty64):
===============================================================

Service files when using xenial64 docker:
--------------
```
cat /etc/systemd/system/docker.service
cat /etc/systemd/system/docker.socket
```

Restart docker via systemd
--------------------------
```
sudo systemctl restart docker
```

Start Registry
--------------
```
cd /docker-registry
docker-compose up
```

Then login on each kubernetes machine to create the /root/.docker/config.json auth hash:
```
docker login https://kmas1.lan
```

##################################################

Ubuntu Xenial 64
========

Manual Docker based Ubuntu Xenial64 Installation:
---------------------

NOTE: The Ansible role should handle the registry build correctly.
The following steps have been adapted from Ubuntu Trusty 64. These SHOULD work for Xenial64:/

To begin using the registry, several steps must be performed. Since we are using self signed SSL certificates, we must share these with our cluster nodes. Docker Registry expects these to exist in a specific location, so the steps below handle that:

```
sudo mkdir -p /etc/docker/certs.d/kmas1.lan
sudo mkdir /usr/local/share/ca-certificates/kmas1.lan

sudo scp vagrant@kmas1.lan:/etc/ssl/kmas1.lan/kmas1.lan.crt /etc/docker/certs.d/kmas1.lan/ca.crt
sudo cp /etc/docker/certs.d/kmas1.lan/ca.crt /usr/local/share/ca-certificates/kmas1.lan/ca.crt

sudo update-ca-certificates

sudo systemctl restart docker

```

Then followed this kubernetes guide for Docker based kubernetes:
[http://kubernetes.io/docs/getting-started-guides/docker/](http://kubernetes.io/docs/getting-started-guides/docker/)


Login to the registry:
----------------------
When I originally had created the kubernetes user, instead of modifying root
```
sudo chown kubernetes:kubernetes /home/kubernetes/.docker/config.json # This is a warning, doesn't seem to affect anything
```
Running commands as root user:
```
docker login https://kmas1.lan
docker pull ubuntu
docker tag ubuntu kmas1.lan/ubuntu
docker push kmas1.lan/ubuntu
```

Manually Install Kubernetes on Trusty 64:
===================

Clone repo and edit config-defaults.sh to install master (example config-default.sh that works is located in roles/kubernetes/templates/working-config-default.sh):
```
sudo su
cd /opt
git clone https://github.com/kubernetes/kubernetes.git
cd /opt/kubernetes/cluster/ubuntu/
vi ./config-default.sh
cd /opt/kubernetes/cluster/
KUBERNETES_PROVIDER=ubuntu ./kube-up.sh
ln -s /opt/kubernetes/cluster/ubuntu/binaries/kubectl /usr/bin/kubectl
kubectl get nodes
```

Dashboard install: (from clusters/ubuntu folder)
```
cd /opt/kubernetes/cluster/ubuntu/
KUBERNETES_PROVIDER=ubuntu ./deployAddons.sh
kubectl get pods --namespace=kube-system
```

Dashboard (supposedly included in kube > 1.2):

```
kubectl cluster-info
```
Output:
```
Kubernetes master is running at http://192.168.0.11:8080
KubeDNS is running at http://192.168.0.11:8080/api/v1/proxy/namespaces/kube-system/services/kube-dns
kubernetes-dashboard is running at http://192.168.0.11:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
```

Kube-system Endpoint:
```
http://192.168.0.11:8080/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/workload?namespace=kube-system
```

Add Wordpress Pod:
==================
```
kubectl run wordpress --image=tutum/wordpress --port=80 --hostport=81
kubectl run wordpress2 --image=tutum/wordpress --port=80 --hostport=82
```

=========================================
   Additional Alternate Installations:
=========================================

Guide specific kubernetes commands:
Running as kubernetes user
```
export K8S_VERSION=$(curl -sS https://storage.googleapis.com/kubernetes-release/release/latest.txt)
export ARCH=amd64
docker run -d     --volume=/:/rootfs:ro     --volume=/sys:/sys:ro     --volume=/var/lib/docker/:/var/lib/docker:rw     --volume=/var/lib/kubelet/:/var/lib/kubelet:rw     --volume=/var/run:/var/run:rw     --net=host     --pid=host     --privileged     gcr.io/google_containers/hyperkube-${ARCH}:${K8S_VERSION}     /hyperkube kubelet         --containerized         --hostname-override=127.0.0.1         --api-servers=http://localhost:8080         --config=/etc/kubernetes/manifests         --cluster-dns=10.0.0.10         --cluster-domain=cluster.local         --allow-privileged --v=2
```

Check Docker processes:
```
docker ps
```

Download kubectl:
-----------------
```
sudo curl -sSL "http://storage.googleapis.com/kubernetes-release/release/v1.2.0/bin/linux/amd64/kubectl" > /usr/bin/kubectl
ls /usr/bin
exit
kubectl get nodes
kubectl run nginx --image=nginx --port=80
docker ps
kubectl expose deployment nginx --port=80
ip=$(kubectl get svc nginx --template={{.spec.clusterIP}})
echo $ip
kubectl get svc nginx --template={{.spec.clusterIP}}
curl 10.0.0.90
lsof -i
sudo lsof -i
docker ps
kubectl cluster-info
kubectl run my-nginx --image=nginx --replicas=2 --port=80 --expose --service-overrides='{ "spec": { "type": "LoadBalancer" } }'
kubectl get po
kubectl get service/my-nginx
curl 10.0.0.137
kubectl delete deployment,service my-nginx
```

Kubernetes Ubuntu Installer (Alternate install version used on Ubuntu 14.04 LTS)
===========================

```
git clone https://github.com/kubernetes/kubernetes.git
sudo service kube-controller-manager start
cd kubernetes/
ls
cd cluster/ubuntu/
ls
vi config-default.sh
```
Testing ssh:
```
ssh root@kmin1.lan
ssh root@kmin2.lan
ssh root@kmin3.lan
```
