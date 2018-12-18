# Creating Highly Available Clusters on some Raspberry Pi

<img src="https://github.com/kubernetes/kubernetes/raw/master/logo/logo.png" width="100">  


---
This tutorial is make for all peoples who wants to try what kubernetes can do  
on a cluster of X Raspberry Pi. Highly Available means that we have multiple  
master. Here, we used 16 Raspberry Pi 3 Model B+.  
Like it's wrote on Kubernetes Documentation, plase notice that:  

"Your clusters must run Kubernetes version 1.13.   
You should also be aware that setting up HA clusters with kubeadm is   
still experimental. You might encounter issues with upgrading your clusters,  
for example. We encourage you to try either approach, and provide feedback."   

Our method is probably not the best, but it's work. We are not perfect, we  
made a lot of mistakes to come to a decent result. If you have a better  
solution for that setup, please make us reports and feedbacks.

---  

## Who are we ?  

This solution is proposed by a group of students in the CyberSecurity Classroom (2018)   
of the Engineering School called "Isen Méditerranée" at Toulon (France).  
All this tutorial is a result of our first project.

## Why Raspberry Pi ?

"Kubernetes is an open source system for managing containerized applications across  
multiple hosts; providing basic mechanisms for deployment, maintenance, and  
scaling of applications."

So, the question is legitimate: Why a Setup on Raspberry Pi ?  
There are many reasons for that:  
- The challenge
- It's experimental, so we learned lot of things about how things work.
- If an application is enough optimize to work on Raspberry Pi (like WebApp or WebSite),
the traffic can be manage with that solution, so you will not be forced to take a server immediatly
- You can learn to manage Kubernetes on local solution, with a very low cost.  

## Let's start !  

### Requirements

- 5 Raspberry Pi (3 B+ in our case) at least
- Same amount of micro-SD cards
- Hypriot OS (1.9 in our case): https://blog.hypriot.com/downloads/
- Our config files (You can try with Kubernetes Documentation too, but with arm, there is some spécifications on .yaml format. Notice them before start)

An external note will come soon too explain all our choice.  
For this README, we will just give our commands to go FROM SCRATCH to our kubernetes System

### First Step: Install OS + Kubernetes

Flash all your cards with Hypriot. You can use Linux commands, flash is a good one.  
If you have windows or if you prefer graphical stuff, please look at solutions like [Etcher](https://www.balena.io/etcher/) or go on your own.  

By default, ssh is fully open on Hypriot installs.  
You can connect your pi via:

    ssh pirate@X.X.X.X (Replace X.X.X.X by Raspberry's IP)  
    Default password: hypriot  

Notice that, if you don't know Raspberry IP, you have many solutions:  
- Try to get a graphic access via HDMI
- Analyse your network with tools like [Angry Ip Scanner](https://angryip.org/) (By default, their hostname is black-pearl)  

Modify the hostname of each pi by something which can be use to identify him.  
For our solution, we have 16 Raspberry with these hostname:  
rasp00,rasp01...rasp15
To modify the default hostname:

    sudo -E -s nano /etc/hostname  
    or  
    sudo -E -s vi /etc/hostname


On each PI you will now need to install Kubernetes.  
If you have too many, write a script or use Ansible can be a good idea  
Please now connect on each pi, and do the followings commands (if you have to sudo, please sudo -E -s):

    apt-get update && apt-get install -y apt-transport-https curl  
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -  
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list  
    deb https://apt.kubernetes.io/ kubernetes-xenial main  
    EOF  
    apt-get update  
    apt-get install -y kubelet kubeadm kubectl  
    apt-mark hold kubelet kubeadm kubectl

Sometimes, a last command is necessary to get the last version:

    apt-get upgrade kubelet kubeadm kubectl

### Second Step: Define your System

Our team have 16 Raspberry Pi. Our design follow these rules:
- All our Pi have fixed IP to 192.168.250.100 to 192.168.250.115
- Our master will be .100 .106 and .112
- .101 will be our load balancer
- They have all an hostname following their IP: rasp00 = .100 rasp12= .112...

So, now, please connect you to your load balancer Pi and install HAProxy:  

    sudo apt-get install haproxy

Edit this file: /etc/haproxy/haproxy.cfg  
Let what is inside by default and down this, we add this:


    frontend k8s-api
        bind 192.168.250.101:6443
        bind 127.0.0.1:6443
        mode tcp
        option tcplog
        default_backend k8s-api

    backend k8s-api
        mode tcp
        option tcplog
        option tcp-check
        balance roundrobin
        default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100

            server rasp00 192.168.250.100:6443 check
            server rasp06 192.168.250.106:6443 check
            server rasp12 192.168.250.112:6443 check


Please replace our Hostname and our IP with yours. (Follow the design setup above)  
Notice that you can replace the IP also with DNS  


### Third Step: Time to Kube  

Now, let's connect to your first master. Create at /home/pirate/ a kubeadm-config.yaml file. Content of the file:

    apiVersion: kubeadm.k8s.io/v1beta1
    kind: ClusterConfiguration
    kubernetesVersion: stable
    apiServer:
      certSANs:
      - "192.168.250.101"
    controlPlaneEndpoint: "192.168.250.101:6443"

Note please than that config comes from the [Kubernetes Documentation](https://kubernetes.io/docs/setup/independent/high-availability/) for the version 1.13.  
It can change with the time so pay attention to the last version of this README.
Note also the indications of Kubernetes Documentation:

- ```kubernetesVersion``` should be set to the Kubernetes version to use. This example uses ```stable```. Note that for our solution, stable seems to be an option to use with precaution.
- ```controlPlaneEndpoint``` should match the address or DNS and port of the load balancer.
- It’s recommended that the versions of ```kubeadm```, ```kubelet```, ```kubectl``` and ```Kubernetes``` match.

Now, let's go for your first kubernetes init. It can be a sensitve point on arm, and if you want to go further on how manage kubernetes on arm, you will often have problem on this step. But if you have following our indications correctly, all will be fine, so let's go:

    sudo kubeadm init --config=kubeadm-config.yaml

After some time, you will get a result like this:

    ...
    ...
    You can now join any number of machines by running the following on each node
    as root:

    kubeadm join 192.168.250.101:6443 --token j04n3m.octy8zely83cy2ts --discovery-token-ca-cert-hash    sha256:84938d2a22203a8e56a787ec0c6ddad7bc7dbd52ebabc62fd5f4dbea72b14d1f

``Please copy this output to a text file, you will need it later.``

To start to manage kubernetes, please go:

    cp /etc/kubernetes/admin.conf /home/pirate  
    sudo chmod 644 /home/pirate/admin.conf  
    export KUBECONFIG=/home/pirate/admin.conf  

You need to make the export in each shell/connection

You can now use command like:

    kubectl get nodes

As you can see, your master is on NotReady state.  
In fact, kubernetes need a pod network add on to work properly. We are on ARM, so the choice is very low.  
Flannel got some unknowns problems with the last updates of kubernetes so we took Weave but with some modifications because Weave have also some problems, but atm, we knowing them and we can solve them.  

    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.WEAVE_NO_FASTDP=1"

Wait a bit, and after a little minute, you will see your first master in ready state. You can also verify the state of Weave thanks to this:

    kubectl get pod -n kube-system -w

### Last Step: In the End

It starts with some files copy. On your first master, execute the following code and don't forget to replace our ip with yours:

    USER=pirate
    CONTROL_PLANE_IPS="192.168.250.106 192.168.250.112"
    for host in ${CONTROL_PLANE_IPS}; do
        scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
        scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
        scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
        scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
        scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
        scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
        scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
        scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
        scp /etc/kubernetes/admin.conf "${USER}"@$host:
    done

Notice than you can put the number of master you want. We chose 3, but you have just to add some CONTROL_PLANE_IPS to this little script and to your load balancer.  
Now, disconnect your first master and go on one of the others. The following proceed have to be repeated on each Master which you want.  
Use this script on each Master (except first one):

    USER=pirate
    mkdir -p /etc/kubernetes/pki/etcd
    mv /home/${USER}/ca.crt /etc/kubernetes/pki/
    mv /home/${USER}/ca.key /etc/kubernetes/pki/
    mv /home/${USER}/sa.pub /etc/kubernetes/pki/
    mv /home/${USER}/sa.key /etc/kubernetes/pki/
    mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/
    mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/
    mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt
    mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key
    mv /home/${USER}/admin.conf /etc/kubernetes/admin.conf

Now take your join command and add this flag --experimental-control-plane  
You will get something like that:
>sudo kubeadm join 192.168.250.101:6443 --token j04n3m.octy8zely83cy2ts --discovery-token-ca-cert-hash sha256:84938d2a22203a8e56a787ec0c6ddad7bc7dbd52ebabc62fd5f4dbea72b14d1f --experimental-control-plane

Now let's make a classic join on all nodes:
>sudo kubeadm join 192.168.250.101:6443 --token j04n3m.octy8zely83cy2ts --discovery-token-ca-cert-hash sha256:84938d2a22203a8e56a787ec0c6ddad7bc7dbd52ebabc62fd5f4dbea72b14d1f

Let's look at the result with:

    kubectl get nodes

You have now An Highly Available Kubernetes Cluster.  
GG !
