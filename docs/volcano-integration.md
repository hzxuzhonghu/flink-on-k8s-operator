# Integration with Volcano for Batch Scheduling

[Volcano](https://github.com/volcano-sh/volcano) is a batch system built on Kubernetes. It provides a suite of mechanisms
currently missing from Kubernetes that are commonly required by many classes
of batch & elastic workloads.
With the integration with Volcano, Flink job and task managers can be scheduled for better scheduling efficiency.

## Prerequisites

## Install Volcano

- Install from provided demo

Run the following 
```bash
kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/master/installer/volcano-development.yaml
```

- Install with advanced settings

Please refer to [Volcano Official Guide](https://volcano.sh/docs/getting-started/)
 
## Install Flink Operator with Volcano enabled

## Installing the Chart

1. Prepare a Flink operator image. Follow the instructions [here](https://github.com/GoogleCloudPlatform/flink-on-k8s-operator/blob/master/docs/developer_guide.md#build-and-push-docker-image) to build and push an image from the source code.

2. Run the bash script `update_template.sh` to update the manifest files in templates from the Flink operator source repo (This step is only required if you want to install from the local chart repo).

3. Register CRD - Don't manually register CRD unless helm install below fails (You can skip this step if your helm version is v3). 
    
    ```bash
   kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/flink-on-k8s-operator/master/config/crd/bases/flinkoperator.k8s.io_flinkclusters.yaml
   ```

4. Finally operator chart can be installed by running:

	```bash
	helm repo add flink-operator-repo https://googlecloudplatform.github.io/flink-on-k8s-operator/
	helm install --name [RELEASE_NAME] flink-operator-repo/flink-operator --set operatorImage.name=[IMAGE_NAME]
	```
    or to install it using local repo with command:

    ```bash
    helm install --name [RELEASE_NAME] . --set operatorImage.name=[IMAGE_NAME] --set enableBatchScheduler=true
    ```


# Create a sample Flink session cluster

Following the guide from [Create a sample Flink cluster](./user_guide.md#create-a-sample-flink-cluster)

Create a sample Flink job cluster with:

```bash
$ kubectl apply -f config/samples/flinkoperator_v1beta1_flinksessioncluster.yaml
```

and verify the pod is up and running with

```bash
$ kubectl get pods,svc -n default | grep "flinksessioncluster"
pod/flinksessioncluster-sample-jobmanager-6d84cd6959-b68dn    1/1     Running   0          21h
pod/flinksessioncluster-sample-taskmanager-7c6b8cdc64-x44r4   2/2     Running   7          21h
service/flinksessioncluster-sample-jobmanager   ClusterIP   10.106.15.205   <none>        6123/TCP,6124/TCP,6125/TCP,8081/TCP   21h
```

verify `job manager` and `task manager` are scheduled by volcano

```bash
$ kubectl get podgroup flink-flinksessioncluster-sample -oyaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  creationTimestamp: "2020-06-18T09:34:01Z"
  generation: 6
  name: flink-flinksessioncluster-sample
  namespace: default
  ownerReferences:
  - apiVersion: flinkoperator.k8s.io/v1beta1
    blockOwnerDeletion: false
    controller: true
    kind: FlinkCluster
    name: flinksessioncluster-sample
    uid: 25bcb47f-e4eb-4619-87ba-c848c42a5d90
  resourceVersion: "1915"
  selfLink: /apis/scheduling.volcano.sh/v1beta1/namespaces/default/podgroups/flink-flinksessioncluster-sample
  uid: 5b7ea6bb-ccf3-483c-8ea3-36007a4aeeb2
spec:
  minMember: 2
  minResources:
    cpu: 400m
    memory: 2Gi
status:
  phase: Running
  running: 
```

As shown above, the podgroup has two pods in running phase and the min required number is 2, that means if the cluster has no enough resources to run both the job manager and task manager, then they are not scheduled.

Also you can check the job manager and task manager's scheduler name is now set to `volcano`

```bash
$ kubectl get pod flinksessioncluster-sample-taskmanager-7c6b8cdc64-x44r4 -ojsonpath={'.spec.schedulerName'}
volcano
```