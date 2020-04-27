# Raspberry Pi with MicroK8s.



## Raspberry PI Overview

### Hardware

* Raspberry Pi 4 Model B with 2GB RAM
* MicroSD Card 32 GB
* Raspberry Pi 15W USB-C Power supply (works better with this)
* Armor Case "BLOCK"
* DisplayPort cable (for initial setup)
* Logitech Wireless Keyboard (for initial setup)



## Install Ubuntu

Instructions which are the basis for these notes:
https://ubuntu.com/blog/single-node-kubernetes-on-raspberry-pi-microk8s-ubuntu


### Create Boot Image

The image from Ubuntu (I used 19.10 64-bit):
https://ubuntu.com/download/raspberry-pi

For writing the image to an MicroSD card:
https://www.balena.io/etcher/



### Boot and configure the image

Use the DisplayPort cable and Keyboard for this part. Afterwards, connect using
SSH over the Wireless connection.

    $ cat /etc/netplan/50-cloud-init.yaml
        network:
            ethernets:
                eth0:
                    dhcp4: true
                    optional: true
            version: 2
            wifis:
                 wlan0:
                     access-points:
                             "X":
                                     password: "Y"
                     dhcp4: true
                     optional: true
    $ sudo netplan apply
    $ ip a
    $ cat /boot/firmware/config.txt
    $ cat /boot/firmware/nobtcmd.txt
        cgroup_enable=... cgroup_enable=memory cgroup_memory=1 ...       <=== add these
    $ sudo apt-get install snapd
    $ sudo reboot


### Configure "ubuntu" user



### System Info

    $ uname -a
        Linux ubuntu 5.3.0-1022-raspi2 #24-Ubuntu SMP Fri Mar 27 21:32:13 UTC 2020 aarch64 aarch64 aarch64 GNU/Linux
    $ lsb_release  -a
        No LSB modules are available.
        Distributor ID: Ubuntu
        Description:    Ubuntu 19.10
        Release:        19.10
        Codename:       eoan
    $ free -h
                      total        used        free      shared  buff/cache   available
        Mem:          1.8Gi       948Mi       320Mi       4.0Mi       579Mi       865Mi
        Swap:            0B          0B          0B
    $ ifconfig
        ...
        wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.24  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::dea6:32ff:fe27:c5a1  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:27:c5:a1  txqueuelen 1000  (Ethernet)
        RX packets 268391  bytes 379681903 (379.6 MB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 60681  bytes 6533026 (6.5 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        ...



## Install MicroK8s

    $ sudo snap install microk8s --classic
    $ sudo usermod -a -G microk8s ubuntu


    $ microk8s.inspect
    $ sudo journalctl -u snap.microk8s.daemon-kubelet
    $ sudo systemctl restart snap.microk8s.daemon-kubelet

    $ microk8s.enable dashboard dns ingress storage
    $ token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
    $ microk8s kubectl -n kube-system describe secret $token
    $ microk8s.status
        microk8s is running
        addons:
        dashboard: enabled
        dns: enabled
        ingress: enabled
        storage: enabled
        helm: disabled
        helm3: disabled
        kubeflow: disabled
        metallb: disabled
        metrics-server: disabled
        rbac: disabled
        registry: disabled

    # Not sure what these were for (could be suggestions from microk8s.inspect)
    $ sudo journalctl -u snap.microk8s.daemon-kubelet
    $ sudo systemctl restart snap.microk8s.daemon-kubelet


### Access the Dashboard

    $ token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
    $ microk8s kubectl -n kube-system describe secret
    $token$ k port-forward -n kube-system service/kubernetes-dashboard 10443:443 --address 0.0.0.0

Navigate to : https://192.168.1.24:10443 and enter the token (from above).



## Setup Ingress for TCP and UDP

Both TCP and UDP traffic can be proxied by the NGINX Ingress Controller which
is included with MicroK8s. To enable this functionality the Ingress Controller
must be patched, and a pair of ConfigMaps created (one for each of TCP and UDP).

    $ git clone https://github.com/trulede/raspberry.microk8s.git
    $ cd raspberry.microk8s/services/ingress
    $ k -n ingress patch daemonset nginx-ingress-microk8s-controller --patch "$(cat ingress-patch.yaml)"
        daemonset.apps/nginx-ingress-microk8s-controller patched
    $ k -n ingress apply -f ingress-configmap.yaml
        configmap/nginx-ingress-tcp-services-conf created
        configmap/nginx-ingress-udp-services-conf created
    $ k -n ingress get daemonset
        NAME                                DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
        nginx-ingress-microk8s-controller   1         1         1       1            1           <none>          6d11h
    $ k -n ingress get configmap
        NAME                                DATA   AGE
        ingress-controller-leader-nginx     0      6d10h
        nginx-ingress-tcp-services-conf     0      14s
        nginx-ingress-udp-services-conf     0      14s
        nginx-load-balancer-microk8s-conf   0      6d10h
    $ k -n ingress get daemonset/nginx-ingress-microk8s-controller -o yaml
        apiVersion: apps/v1
        kind: DaemonSet
        metadata:
          name: nginx-ingress-microk8s-controller
          namespace: ingress
         spec:
          ...
          template:
            ...
            spec:
              containers:
              - args:
                - /nginx-ingress-controller
                - --configmap=$(POD_NAMESPACE)/nginx-load-balancer-microk8s-conf
                - --tcp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-tcp-services-conf
                - --udp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-udp-services-conf
                - --publish-status-address=127.0.0.1


More information can be found at these links:
* https://kubernetes.github.io/ingress-nginx/user-guide/exposing-tcp-udp-services/
* https://minikube.sigs.k8s.io/docs/tutorials/nginx_tcp_udp_ingress/



## Use the K8s (single node) Cluster

Before setting up any of the following services on your Raspberry Pi run the
following commands:

    $ git clone https://github.com/trulede/raspberry.microk8s.git
    $ cd raspberry.microk8s/services
    $ k apply -f namespace.yaml


### Redis

First install a Redis Container into the "pi" namespace.

    $ cd raspberry.microk8s/services/redis
    $ k apply -f redis.yaml
    $ k -n pi get services,pods
        NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
        service/redis   ClusterIP   10.152.183.122   <none>        6379/TCP   45s

        NAME                         READY   STATUS    RESTARTS   AGE
        pod/redis-6fbc78dc54-9xpvv   1/1     Running   0          44s


Then patch the Ingress Controller and TCP ConfigMap to enable the proxy of
incoming TCP connections send them to the Redis Service.

    $ k -n ingress patch daemonset nginx-ingress-microk8s-controller --patch "$(cat redis-patch-ingress-daemonset.yaml)"
        daemonset.apps/nginx-ingress-microk8s-controller patched
    $ k -n ingress patch configmap nginx-ingress-tcp-services-conf --patch "$(cat redis-patch-ingress-tcp-configmap.yaml)"
        configmap/nginx-ingress-tcp-services-conf patched


Finally install the Redis CLI and check things are working.

    $ sudo apt install redis-tools
    $ k -n pi get service
        NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
        redis   ClusterIP   10.152.183.238   <none>        6379/TCP   10h
    $ redis-cli
        127.0.0.1:6379> set foo bar
        OK
        127.0.0.1:6379> get foo
        "bar"
        127.0.0.1:6379> keys *
        1) "foo"
        127.0.0.1:6379> exit


Alternative method using port-forward (rather than then Ingress Controller).

    $ k -n pi port-forward deployment/redis 6379:6379 &
        [1] 18995
        Forwarding from 127.0.0.1:6379 -> 6379
        Forwarding from [::1]:6379 -> 6379
    $ redis-cli
        Handling connection for 6379
        127.0.0.1:6379> set foo bar
        OK
        127.0.0.1:6379> get foo
        "bar"
        127.0.0.1:6379> keys *
        1) "foo"
        127.0.0.1:6379> exit
    $ fg 1
        microk8s.kubectl -n pi port-forward deployment/redis 6379:6379
        <CTRL C>



### Minecraft

Original from: https://github.com/itzg/docker-minecraft-bedrock-server

> Warning: The Minecraft Server deployment to Raspberry Pi does not work. If
> you decide to run it and find your Pi becomes unresponsive, the easiest way
> to recover the system is to restart the Pi and quickly delete the "pi"
> namespace: k delete namespaces pi

The following commands install a Minecraft Bedrock Server with a 1 GB volume
and configuration for the Ingress Controller.

    $ cd raspberry.microk8s/services/minecraft
    $ k apply -f minecraft.yaml
        persistentvolumeclaim/minecraft created
        deployment.apps/minecraft created
        service/minecraft created
    $ k -n pi get services,pods
        NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
        service/redis   ClusterIP   10.152.183.122   <none>        6379/TCP   45s

        NAME                         READY   STATUS    RESTARTS   AGE
        pod/redis-6fbc78dc54-9xpvv   1/1     Running   0          44s
    $ k get pvc
        NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        AGE
        minecraft   Bound    pvc-8fdca5bd-8dc4-4224-a85f-382d9cc271f0   1Gi        RWO            microk8s-hostpath   10m
    $ k -n ingress patch daemonset nginx-ingress-microk8s-controller --patch "$(cat minecraft-patch-ingress-daemonset.yaml)"
        daemonset.apps/nginx-ingress-microk8s-controller patched
    $ k -n ingress patch configmap nginx-ingress-udp-services-conf --patch "$(cat minecraft-patch-ingress-udp-configmap.yaml)"
        configmap/nginx-ingress-udp-services-conf patched
