---
title: Instalar Kubernetes a través de Ansible
date: 2023-06-20 18-01-00
categories: [Proyecto Itegrado]
tags: [ansible,kubernetes]
---

## Introducción

En esta guía vamos a explicar como lanzar un playbook de Ansible para que despliegue y configure un cluster de kubernetes

## Pasos

### Playbooks para instalación y configuración

1. Crea un archivo yaml llamado *users.yaml* y pega la siguiente configuración

    El primer playbook que se va a lanzar es users.yaml el cual va a crear un usuario llamado kube que se va a encargar de gestionar los siguientes playbooks de Ansible:

    ```yaml
    - hosts: 'workers, masters'
    become: yes

    tasks:
        - name: create the kube user account
        user: name=kube append=yes state=present createhome=yes shell=/bin/bash

        - name: allow 'kube' to use sudo without needing a password
        lineinfile:
            dest: /etc/sudoers
            line: 'kube ALL=(ALL) NOPASSWD: ALL'
            validate: 'visudo -cf %s'

        - name: set up authorized keys for the kube user
        authorized_key: user=kube key="{{item}}"
        with_file:
            - ~/.ssh/kube-ansible.pub
    ```

2. Crea un archivo llamado *crear-master.yaml* y pega la siguiente configuración

    ```yaml
    - hosts: masters
    become: yes
    tasks:
        - name: initialize the cluster
        shell: |
                sudo kubeadm reset -f
                sudo rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/*
                kubeadm init --pod-network-cidr=10.244.0.0/16
        args:
            chdir: $HOME
            creates: cluster_initialized.txt

        - name: create .kube directory
        become: yes
        become_user: kube
        file:
            path: $HOME/.kube
            state: directory
            mode: 0755

        - name: copies admin.conf to user's kube config
        copy:
            src: /etc/kubernetes/admin.conf
            dest: /home/kube/.kube/config
            remote_src: yes
            owner: kube

        - name: install Pod network
        become: yes
        become_user: kube
        shell: kubectl apply -f https://docs.projectcalico.org/archive/v3.20/manifests/calico.yaml
        args:
            chdir: $HOME
        - name: Get the token for joining the worker nodes
        become: yes
        become_user: kube
        shell: kubeadm token create  --print-join-command
        register: kubernetes_join_command

        - name: debugging
        debug:
            msg: "{{ kubernetes_join_command.stdout }}"

        - name: Copy join command to local file.
        become: yes
        local_action: copy content="{{ kubernetes_join_command.stdout_lines[0] }}" dest="/tmp/kubernetes_join_command" mode=0777
    ```

3. Crea un archivo llamado *crear-master.yaml* y pega la siguiente configuración

    ```yaml
    - hosts: workers
    become: yes
    gather_facts: yes

    tasks:
    - name: Copy join command from Ansiblehost to the worker nodes.
        become: yes
        copy:
        src: /tmp/kubernetes_join_command
        dest: /tmp/kubernetes_join_command
        mode: 0777

    - name: Join the Worker nodes to the cluster.
        become: yes
        command: sh /tmp/kubernetes_join_command
        register: joined_or_not
    ```

4. Crea un archivo llamado *instalar-kubernetes.yaml* y pega la siguiente configuración

    ```yaml
    ---
    - hosts: "masters, workers"
    remote_user: ubuntu
    become: yes
    become_method: sudo
    become_user: root
    gather_facts: yes
    connection: ssh

    tasks:
        - name: Create containerd config file
        file:
            path: "/etc/modules-load.d/containerd.conf"
            state: "touch"

        - name: Add conf for containerd
        blockinfile:
            path: "/etc/modules-load.d/containerd.conf"
            block: |
                overlay
                br_netfilter

        - name: modprobe
        shell: |
                sudo modprobe overlay
                sudo modprobe br_netfilter


        - name: Set system configurations for Kubernetes networking
        file:
            path: "/etc/sysctl.d/99-kubernetes-cri.conf"
            state: "touch"

        - name: Add conf for containerd
        blockinfile:
            path: "/etc/sysctl.d/99-kubernetes-cri.conf"
            block: |
                    net.bridge.bridge-nf-call-iptables = 1
                    net.ipv4.ip_forward = 1
                    net.bridge.bridge-nf-call-ip6tables = 1

        - name: Apply new settings
        command: sudo sysctl --system

        - name: install containerd
        shell: |
                sudo apt-get update && sudo apt-get install -y containerd
                sudo mkdir -p /etc/containerd
                sudo containerd config default | sudo tee /etc/containerd/config.toml
                sudo systemctl restart containerd

        - name: disable swap
        shell: |
                sudo swapoff -a
                sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

        - name: install and configure dependencies
        shell: |
                sudo apt-get update && sudo apt-get install -y apt-transport-https curl
                curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

        - name: Create kubernetes repo file
        file:
            path: "/etc/apt/sources.list.d/kubernetes.list"
            state: "touch"

        - name: Add K8s Source
        blockinfile:
            path: "/etc/apt/sources.list.d/kubernetes.list"
            block: |
                deb https://apt.kubernetes.io/ kubernetes-xenial main

        - name: install kubernetes
        shell: |
                sudo apt-get update
                sudo apt-get install -y kubelet=1.26.1-00 kubeadm=1.26.1-00 kubectl=1.26.1-00
                sudo apt-mark hold kubelet kubeadm kubectl
    ```

### Playbook maestro

A continuación vamos a crear un playbook maestro que lance los otros en orden para instalar y configurar el cluster de kubernetes

1. Creamos el playbook maestro llamado *kubernetes-main.yaml* y pegamos lo siguiente

    ```yaml
    ---
    - name: Crear usuario kube
    ansible.builtin.import_playbook: ./users.yaml
    - name: instalar kubernetes
    ansible.builtin.import_playbook: ./instalar-kubernetes.yaml
    - name: crear nodo master
    ansible.builtin.import_playbook: ./crear-master.yaml
    - name: Crear nodos workers
    ansible.builtin.import_playbook: ./crear-workers.yaml
    ```

2. Lanzamos el playbook maestro con Ansible

    ```bash
    ansible-playbook kubernetes-main.yaml -f 10
    ```
