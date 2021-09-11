---
layout: post
title: Home media server with k8s
date: 2021-05-08 10:45 +0400
category: 
author:
tags: [k8s, Home media server]
summary: This post is to discuss how we can create home server with k8s
---

Loved to have exclusive media server for my home, previously used Emby application for the purpose and pretty impressed about it. I had NUC device dedicated to that and with the help of docker i have somehow managed to run the application, but the challenge was using graphics card for Emby, Since NUC have small form factor it does not well support adding graphics card. So i decided to give a chance to device abstraction by using k8s. My thought is create a cluster with NUC as master and join personal laptop to it, which has graphics card, And configure the emby application to have more affinity towards graphics card machine, while other supporting software on NUC. I know managing a k8s cluster for media server is a over kill, but who don't loves having a challenge and learning opportunity.

# Install K8s on NUC
1. Become root and update and upgrade NUC

    ```bash
    sudo -i
    apt-get update && apt-get upgrade -y
    ```
2. Install vim if you haven't
   
   ```bash
   apt-get install -y vim
   ```
3. Install Docker
    ```bash
    apt-get install -y docker.io
    ```
4. k8s is not available in Ubuntu's default repository, so add it manually

    ```bash
    vim /etc/apt/sources.list.d/kubernetes.list
    ```
    add below line to the file
    ```vim
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    ```
    Add a GPG key for the packages. This will return OK.
    ```bash
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    ```
    update with the new repo, which will download repo info.

5. Install k8s software.
    ```bash
    apt-get install -y kubeadm=1.21.0-00 kubelet=1.21.0-00 kubectl=1.21.0-00
    apt-mark hold kubelet kubeadm kubectl
    ```
6. Create configuration file for cluster
    1. Create alias for the master node
            
        ```bash
        vim /etc/hosts

        add this line, by replacing ip address with your server ip.
        <ipaddress of the master> k8smaster
        ```
   2. Configure networking settings
        There can be only one pos network per cluster, we will use Calico as a network plugin. Currently Calico does not deploy using CNI by default.
        
        ```bash
        wget https://docs.projectcalico.org/manifests/calico.yaml
        ```
        
        Adjust the network CIDR as per requirement.
        
        ```bash
        vim calico.yaml
        ```
        
        There are many different configuration settings in this file. The **CALICO_IPV4POOL_CIDR** must match the value given to **kubeadm init** in the following step, and avoid conflicts with existing IP ranges of the nodes.

        ```yaml
        
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "192.168.1.0/16"

        ```

   3. Create cluster config

        ```bash
        vim kubeadm-config.yaml
        ``` 
       
       add the following settings to the file.
       
       ```yaml
        apiVersion: kubeadm.k8s.io/v1beta2
        kind: ClusterConfiguration
        kubernetesVersion: 1.21.0
        controlPlaneEndpoint: "k8smaster:6443"  #<-- Use the node alias 
        networking:
          podSubnet: 192.168.1.0/16             #<-- Match the IP range from the Calico config file
       ```

7. Initialize the master.
    
    ```bash
    kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
    ```

    This may return error as docker not running
    ```bash
    ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?

    # Start the docker daemon
    systemctl start docker.service

    # Initialize again
    kubeadm init --config=kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
    ```

    There may be another error can happen if swap is on

    ```bash
    sudo swapoff -a
    ``` 
   

8. Allow a non-root user admin level access to the cluster.

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    ```

9. Apply the network configuration

    ```bash
    sudo cp /root/calico.yaml .
    kubectl apply -f calico.yaml
    ```

10. Enable bash auto-completion.

    ```bash
    sudo apt-get install bash-completion -y

    source <(kubectl completion bash)

    echo "source <(kubectl completion bash)" >> $HOME/.bashrc
    ```