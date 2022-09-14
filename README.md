# wordpress-demo
#Setup a wordpress on k8s and exposing through itstio 
#system requirement Ubuntu 20.04.4 LTS (Focal Fossa)
#Step 1 
#Seting up k8s cluster   (here we  use k3d )
1 what is k3d ?
k3d is a lightweight wrapper to run k3s (Rancher Lab’s minimal Kubernetes distribution) in docker.
k3d makes it very easy to create single- and multi-node k3s clusters in docker, e.g. for local development on Kubernetes.

#k3d installation 
official site link : https://k3d.io/v5.4.6/
here we have installed k3d version v5.4.6

root@shafeeq-c2s:/home/shafeeq/demo/wordpress# k3d --version
k3d version v5.4.6
k3s version v1.24.4-k3s1 (default)

#create k3d multi cluster setup & istio (service mesh) 
k3d cluster create my-multinode-cluster --servers 1 --agents 3 --port 9080:80@loadbalancer --port 9443:443@loadbalancer --api-port 6443 --k3s-server-arg '--no-deploy=traefik'

#command argument explanations
--servers 1 simply means there will be one server node for the control plane.
--agents 3 simply means we will have 3 nodes to run containers on
--port 9080:80@loadbalancer simply means that the load balancer (in docker, which is exposed), will forward requests to port 9080 to 80 in the k8 cluster, you can check this out after creation by running docker ps
— port 944:443@loadbalancer same as above, just for HTTPS (later)
--api-port 6443 the k8 API port will be port 6443 instead of randomly generated
--k3s-server-arg '--no-deploy=traefik' simply means k3d will not deploy the Traefik v1 ingress controller

results : 
root@shafeeq-c2s:/home/shafeeq/demo/wordpress/istio-1.15.0# docker ps
CONTAINER ID   IMAGE                            COMMAND                  CREATED       STATUS       PORTS                                                                                                    NAMES
cc5315052f70   ghcr.io/k3d-io/k3d-proxy:5.4.6   "/bin/sh -c nginx-pr…"   10 days ago   Up 2 hours   0.0.0.0:6443->6443/tcp, 0.0.0.0:9080->80/tcp, :::9080->80/tcp, 0.0.0.0:9443->443/tcp, :::9443->443/tcp   k3d-my-multinode-cluster-serverlb
5e60a22c1be3   rancher/k3s:v1.24.4-k3s1         "/bin/k3s agent"         10 days ago   Up 2 hours                                                                                                            k3d-my-multinode-cluster-agent-2
895ce0a2ce54   rancher/k3s:v1.24.4-k3s1         "/bin/k3s agent"         10 days ago   Up 2 hours                                                                                                            k3d-my-multinode-cluster-agent-1
1e8f501bb6fa   rancher/k3s:v1.24.4-k3s1         "/bin/k3s agent"         10 days ago   Up 2 hours                                                                                                            k3d-my-multinode-cluster-agent-0
240af9c7d855   rancher/k3s:v1.24.4-k3s1         "/bin/k3s server --d…"   10 days ago   Up 2 hours                                                                                                            k3d-my-multinode-cluster-server-0

#k3d cluster list
root@shafeeq-c2s:/home/shafeeq/demo/wordpress# k3d cluster list
NAME                   SERVERS   AGENTS   LOADBALANCER
my-multinode-cluster   1/1       3/3      true





####3Deploying Istio
official link :  https://istio.io

To download the latest version, run the following… (It can take a little while, that or my wifi was freaking out…), this will download the archive and extract it for you…
curl -L https://istio.io/downloadIstio | sh -
once completed, move the folder to the desired location and navigate to it as follows… (Make sure to check which version you have)
cd path/to/istio-1.15.0
eg: 
root@shafeeq-c2s:/home/shafeeq/demo/wordpress/istio-1.15.0# ls 
bin  LICENSE  manifests  manifest.yaml  README.md  samples  tools

then run the following to add it to your path…
export PATH=$PWD/bin:$PATH

check the istio version 
istioctl version
istioctl profile list # Will list available profiles

# Installing the Default Profile


istioctl install --set profile=default

resutlt like :

This will install the Istio default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete

To enable the automatic injection of Envoy sidecar proxies, run the following… (Otherwise you will need to do this manually when you deploy your applications)

#kubectl label namespace default istio-injection=enabled

result : 
root@shafeeq-c2s:/home/shafeeq/demo/wordpress/istio-1.15.0# kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                   PORT(S)                                      AGE
istiod                 ClusterIP      10.43.222.99    <none>                                        15010/TCP,15012/TCP,443/TCP,15014/TCP        10d
istio-ingressgateway   LoadBalancer   10.43.177.186   172.21.0.2,172.21.0.3,172.21.0.4,172.21.0.6   15021:31280/TCP,80:32612/TCP,443:32567/TCP   10d

##deploy word press on k8s
  WordPress is a popular open-source CMS(content management system) software. Many organizations are using WordPress. Why people love WordPress. Because it is open-source software and developed using PHP which is easy to learn. And customization is also very easy if you know the basics of PHP. Many developers are developing both free and paid themes and plugins for WordPress. Installing the themes and plugins is very easy. Just one click to install and uninstall. So you will get plenty of options for your CMS when you use WordPress.
  
  #for mysql deployment  
  1. Create Secret for MySQL

2. Create PVC for MySQL

3. Create Deployment for MySQL

4. Create Service for MySQL
  
  result : 
  kubectl apply -f secrets.yaml
  kubectl apply -f mysql-deployment.yaml
  
  here we usidng mysql:5.6 docker base image from docker hub
  mysql mount path : /var/lib/mysql
  
  #for wordpress application deployment 
>>  Create PVC for WordPress
2. Create Deployment for WordPress

3. Create Service for WordPress
  result : 
  root@shafeeq-c2s:/home/shafeeq/demo/wordpress# kubectl apply -f wordpress-deployment.yaml
service/wordpress configured
persistentvolumeclaim/wp-pv-claim unchanged
deployment.apps/wordpress unchanged
  
  here we are using wordpress:4.8-apache docker base  image from docker hub
  and documentarty root path is /var/www/html
  
  ## now wordpress deployment is done 
  
  root@shafeeq-c2s:/home/shafeeq/demo/wordpress# kubectl get deployments 
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
wordpress         1/1     1            1           24h
wordpress-mysql   1/1     1            1           24h

  
  
  
  ### exposing wordpress site with istio using gatway service 
 kubectl apply -f wordpress-gateway.yaml
  
  result :   
  root@shafeeq-c2s:/home/shafeeq/demo/wordpress# kubectl apply -f wordpress-gateway.yaml
gateway.networking.istio.io/wordpress-gateway configured
virtualservice.networking.istio.io/wordpress configured

  
 #check the wordpress deployment external ip 
  kubectl get svc
  
  eg : 
  root@shafeeq-c2s:/home/shafeeq/demo/wordpress# kubectl get svc
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP                        PORT(S)        AGE
kubernetes        ClusterIP      10.43.0.1      <none>                             443/TCP        10d
wordpress-mysql   ClusterIP      None           <none>                             3306/TCP       10d
wordpress         LoadBalancer   10.43.141.98   172.21.0.2,172.21.0.3,172.21.0.4   80:31784/TCP   10d


  
  now our wordpress site is up and running call on your local browser with  
  
  http://172.21.0.2/31784
  
  



