---
title: Desplegar el servicio de wiki a través de Ansible
date: 2023-06-20 18-03-00
categories: [Proyecto Itegrado]
tags: [ansible,kubernetes,jekyll,apache,wiki]
---

## Introducción

En este apartado vamos a ver los distintos manifiestos que usamos en el proyecto para el despliegue de servicios en el cluster de kubernetes

### Manifiestos

1. Creación del Storage Class
    El storageClass va a ser una etiqueta que identifique el tipo de volumen persistente y su controlador, en este caso, un controlador local

    **storage-class-wiki.yaml**

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
    name: manual
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ```

2. Creación de Volúmenes Persistentes PV (Persistent Volume)

    Los volumenes persistentes nos van a permitir crear un directorio con la información de la página web de la wiki que luego van a servir los pods de apache

    **pv-wiki-site.yaml**

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: pv-wiki-site
    labels:
        type: local
        use: wiki-site
    spec:
    storageClassName: manual
    capacity:
        storage: 5Gi
    volumeMode: Filesystem
    accessModes:
        - ReadWriteOnce
    persistentVolumeReclaimPolicy: Delete
    local:
        path: /wiki/_site
    nodeAffinity:
        required:
        nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
            operator: In
            values:
            - esfhblcub01
            - esfhblcub02
            - esfhblcub03
    ```

3. Creación de Persistent Volume Claim (pvc)

    En este paso creamos un manifiesto que reclama para un servicio en concreto un volumen persistente en concreto

    **pvc-wiki-site.yaml**

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: pvc-wiki-site
    spec:
    storageClassName: manual
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
        storage: 5Gi
    selector: 
        matchLabels: 
        type: local
        use: wiki-site
    ```

4. Creación del manifiesto de despliegue

    En este manifiesto vamos a declarar el pod que queremos, los contenedores que va a tener, los puertos vamos a abrir en cada tipo de contenedor y los volumenes que vamos a montar en los contenedores

    **deployment-apache.yaml**

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: wiki-deployment
    labels:
        app: wiki
    spec:
    replicas: 3
    selector:
        matchLabels:
        app: wiki
    template:
        metadata:
        labels:
            app: wiki
        spec:
        containers:
        - name: httpd
            image: httpd:latest
            ports:
            - containerPort: 80
            volumeMounts:
            - name: wiki-site
            mountPath: "/usr/local/apache2/htdocs"
        volumes:
            - name: wiki-site
            persistentVolumeClaim:
                claimName: pvc-wiki-site
    ```

### Playbook

Ahora solo falta un playbook que despliegue los manifiestos en el master del cluster

```yaml
- hosts: 'workers, masters'

  tasks:
    - name: Crear Persistent volume para apache
      shell: kubectl apply -f ./pv-wiki-site.yaml

    - name: Crear Persistent Volume Claim para apache
      shell: kubectl apply -f ./pvc-wiki-site.yaml

    - name: Desplegar pods
      shell: kubectl apply -f ./deployment-jekyll-apache.yaml

    - name: Crear y desplegar Load Balancer
      shell: kubectl apply -f ./service-balancer-wiki.yaml

```
