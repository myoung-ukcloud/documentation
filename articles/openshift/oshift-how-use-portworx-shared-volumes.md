---
title: How to use Portworx shared volumes
description: Explains how to use shared volumes in a Portworx integrated OpenShift cluster
services: openshift
author: Ben Bacon
reviewer:
lastreviewed: 16/10/2019

toc_rootlink: How To
toc_sub1: OpenShift v3.x
toc_sub2: Portworx
toc_sub3:
toc_sub4:
toc_title: Use Portworx shared volumes
toc_fullpath: How To/Use Portworx with OpenShift/oshift-how-use-portworx-shared-volumes.md
toc_mdlink: oshift-how-use-portworx-shared-volumes.md
---

# How to use Portworx shared volumes

## Overview

Portworx is a cloud-native storage solution that is now available as an integration with our OpenShift offering. This article describes how to create shared volumes.

### Intended audience

This article assumes you have access to a Portworx integrated OpenShift 3.11 cluster (or later) and that you have cluster-admin rights. It also assumes familiarity with `oc`, the OpenShift command-line client.

If you're interested in a free 30 day trial of Portworx, raise a Service Request via the [My Calls](https://portal.skyscapecloud.com/support/ivanti) section of the UKCloud Portal.

## Shared volumes

Portworx enables persistent volumes to be mounted to multiple containers across multiple nodes.

## Storage classes

We create a number of storage classes when provisioning an OpenShift cluster with the Portworx integration. The following example shows a storage class from which shared persistent volume claims (pvc) with a single replica can be provisioned:

```none
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
    name: portworx-repl1-shared
provisioner: com.openstorage.pxd
parameters:
   repl: "1"
   shared: "true"
```

If you want to use different attributes to the pre-provisioned storage classes, include the following values within the spec:

```none
provisioner: com.openstorage.pxd
parameters:
	shared: "true"
```

> [!IMPORTANT]
> Within OpenShift, Portworx shared volumes make use of the Container Storage Interface (CSI) via the `com.openstorage.pxd` provisioner. OpenShift 3.11 uses v0.3 of the CSI spec, which is a beta release, so use of shared volumes is not recommended within a production environment.

## Creating a persistent volume claim

The following example shows a persistent volume claim (pvc) spec using a shared storage class. Note the `ReadWriteMany` access mode:

```none
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: pvc-shared
   annotations:
     volume.beta.kubernetes.io/storage-class: portworx-repl1-shared
spec:
   accessModes:
     - ReadWriteMany
   resources:
     requests:
       storage: 10Gi
```

Once you've created the pvc, verify that its status shows as `Bound` to confirm it has been provisioned by Portworx:

```none
$ oc get pvc
NAME             STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS            AGE
pvc-shared   Bound     pvc-3654b857-efa1-11e9-8422-fa163e52fd0e   10Gi       RWX            portworx-repl1-shared   1m
```

You can also verify that the persistent volume backing the pvc has been created:

```none
$ oc get pv $(oc get pvc pvc-shared -o custom-columns=VOLUME:.spec.volumeName --no-headers)
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                     STORAGECLASS            REASON    AGE
pvc-3654b857-efa1-11e9-8422-fa163e52fd0e   10Gi       RWX            Delete           Bound     portworxdemo/pvc-shared   portworx-repl1-shared             2m
```

## Verifying a shared volume

You can verify that a shared persistent volume is working as expected by creating two deployments that schedule to different nodes and mount the same volume. The following example assumes that you've created a shared volume named `pvc-shared`.

> [!TIP]
> The below file contains two separate [Deployments](https://docs.openshift.com/container-platform/3.11/dev_guide/deployments/how_deployments_work.html#creating-a-deployment-configuration), as can be identified by the YAML document separator `---`. Each spec is near-identical, with the only difference being the command that the pod continually loops.
>
> [```nodeAffinity```](https://docs.openshift.com/container-platform/3.11/admin_guide/scheduling/node_affinity.html#admin-guide-configuring-affinity) is an affinity type used to determine which node a pod should be scheduled to, in this instance we only want the pod to be scheduled to compute nodes so the ```requiredDuringSchedulingIgnoredDuringExecution``` rule is used to check that a node contains the ```node-role.kubernetes.io/compute``` label, which is a label only applied to nodes with the compute role.
>
> [```podAntiAffinity```](https://docs.openshift.com/container-platform/3.11/admin_guide/scheduling/pod_affinity.html#admin-guide-sched-affinity-pod-config) is used to keep pods separate from one another. For each deployment the created pod is labelled with: ```app: portworx-sharedvolumedemo```. The ```preferredDuringSchedulingIgnoredDuringExecution``` rule then checks when scheduling a new pod whether an existing pod has already been scheduled to the same node with the previous stated label. If it does, an alternate node will be selected if there is sufficient available resources.
>
> You can view OpenShift object definitions using the [```oc explain```](https://blog.openshift.com/oc-command-newbies) command. More specifically to identify fields under the Deployment spec key, you would use: ```oc explain deployment.spec```.

1. On your machine, create a file named `sharedvolumedemo.yaml` with the following contents:

    ```none
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: portworx-sharedvolumedemo-a
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: portworx-sharedvolumedemo
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: node-role.kubernetes.io/compute
                    operator: Exists
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - portworx-sharedvolumedemo
                  topologyKey: kubernetes.io/hostname
          volumes:
          - name: shared-data
            persistentVolumeClaim:
              claimName: pvc-shared
          containers:
          - name: debian-container
            image: debian
            volumeMounts:
            - name: shared-data
              mountPath: /pod-data
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                cpu: "250m"
              requests:
                cpu: "250m"
            command: ["/bin/sh"]
            args:
              - "-c"
              - >
                while true; do
                  sleep 300;
                done
    ---
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: portworx-sharedvolumedemo-b
    spec:
      replicas: 1
      template:
        metadata:
          labels:
            app: portworx-sharedvolumedemo
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: node-role.kubernetes.io/compute
                    operator: Exists
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - portworx-sharedvolumedemo
                  topologyKey: kubernetes.io/hostname
          volumes:
          - name: shared-data
            persistentVolumeClaim:
              claimName: pvc-shared
          containers:
          - name: debian-container
            image: debian
            volumeMounts:
            - name: shared-data
              mountPath: /pod-data
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                cpu: "250m"
              requests:
                cpu: "250m"
            command: ["/bin/sh"]
            args:
              - "-c"
              - >
                while true; do
                  date >> /pod-data/sharedfile;
                  echo Hello from the second pod >> /pod-data/sharedfile;
                  sleep 1;
                done
    ```

2. You can now create the deployments with the command: `oc create -f sharedvolumedemo.yaml`, which should output the following:

    ```none
    deployment.extensions/portworx-sharedvolumedemo-a created
    deployment.extensions/portworx-sharedvolumedemo-b created
    ```

3. Once the deployments have been created, retrieve the pod names that have been created by the ReplicaSet and verify that these have been scheduled to different nodes:

    ```none
    $ oc get pods -o wide
    NAME                                           READY     STATUS    RESTARTS   AGE       IP            NODE                            NOMINATED NODE
    portworx-sharedvolumedemo-a-5c5f975f5-sbphh    1/1       Running   0          38s       10.128.4.8    worker-tenant-m-2.portworx-demo   <none>
    portworx-sharedvolumedemo-b-57896c87dd-t55vs   1/1       Running   0          38s       10.131.0.10   worker-tenant-m-0.portworx-demo   <none>
    ```

4. The `b` pod writes the current datetime and a message to a file every minute on the mounted shared volume. With the `a` pod, you can tail this same file on the shared volume to verify that the volume is mounted to multiple pods across multiple nodes:

    ```none
    $ date
    Wed Oct 16 11:36:45 UTC 2019
    $ oc exec portworx-sharedvolumedemo-a-5c5f975f5-sbphh -- /usr/bin/tail -n 4 /pod-data/sharedfile
    Wed Oct 16 11:36:45 UTC 2019
    Hello from the second pod
    Wed Oct 16 11:36:46 UTC 2019
    Hello from the second pod
    ```

## Further reading

<https://docs.portworx.com/concepts/shared-volumes/>

## Feedback

If you find a problem with this article, click **Improve this Doc** to make the change yourself or raise an [issue](https://github.com/UKCloud/documentation/issues) in GitHub. If you have an idea for how we could improve any of our services, send an email to <feedback@ukcloud.com>.
