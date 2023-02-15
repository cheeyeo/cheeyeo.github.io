---
layout: post
show_meta: true
title: Using Flink High Availability with flink operator
header: Using Flink High Availability with flink operator
date: 2022-12-02 00:00:00
summary: How to enable and use flink high availability
categories: datascience beam flink pyflink kubernetes kind
author: Chee Yeo
---

[Checkpoints vs savepoints]: https://nightlies.apache.org/flink/flink-docs-release-1.16/docs/ops/state/checkpoints_vs_savepoints/

[Documentation on Flink HA]: https://cwiki.apache.org/confluence/display/FLINK/FLIP-144:+Native+Kubernetes+HA+for+Flink#FLIP144:NativeKubernetesHAforFlink-ConfigMap

[Flink 1.15.2 configuration]: https://nightlies.apache.org/flink/flink-docs-release-1.15/docs/deployment/config/

[Flink Checkpoints]: https://nightlies.apache.org/flink/flink-docs-master/docs/ops/state/checkpoints/

Flink supports high-availability mode for both standalone and native Kubernetes via the Flink operator. This article aims to explain the purpose of HA and how to configure and run it in a local kind cluster.

A flink cluster has only 1 JobManager running at a given point in time. This presents as single point of failure. If the jobmanager fails currently running jobs within the cluster will also fail and have to be restarted from scratch.

Enabling HA allows the cluster to recover from such failures and ensures that streaming jobs especially can resume from its last known state via checkpoints.

In a previous blog post, I mentioned `savepoints` and how we can resume a job from it. `checkpoints` is a mechanism provided via the HA service. When HA is enabled, for any streaming job, Flink will make regular backups of the job's state via `checkpoints` which allows you to resume the job from in event of a cluster failure.

The main difference between `savepoints` and `checkpoints` is that the former is triggred by the user while the other is managed entirely by Flink.

The main purpose of having two complementary systems is that `checkpoints` provide fast recoverable state in the event of cluster failures such as job manager or task manager pods being killed whereas `savepoints` allow for more portability and is intended for long-term uses such as Flink versions upgrade and changes to job properties.

In HA mode, we can have more than 1 job manager pod running concurrently and only 1 of them is selected as the Leader via the leader election service.

In this post, we are using the `flink-operator` to setup our session cluster. The default setting creates a stateful set of the jobmanager and the number of replicas is not configurable at this point. However, running a replicaset means there will always be at least 1 job manager pod running so it serves this use case.

To enable HA, the following conditions must be met:

* Only use local storage for the high availability checkpoints. In the kind cluster config, we can mount an additional local volume and reference it in a persistent volume. The mounted volume must also have the ownership of `9999:9999`

* A service account which has permissions to edit configmaps. HA stores information on the cluster state such as the current jobmanager in these configmaps.

To create the HA volume I mounted a local volume in /tmp/flink-k8s-example on localhost to /flink-k8s-example on the node in the kind cluster config:
{% highlight yaml linenos %}
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  extraMounts:
    - hostPath: /tmp/artifacts
      containerPath: /artifacts
    - hostPath: /tmp/flink-k8s-example
      containerPath: /flink-k8s-example
{% endhighlight %}

Next we create the PV, and PVC for the mounted volume:
{% highlight yaml linenos %}
apiVersion: v1
kind: PersistentVolume
metadata:
  name: flink-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /flink-k8s-example/
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: flink-shared-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  volumeName: flink-pv
  resources:
    requests:
      storage: 1Gi
{% endhighlight %}

The PV mounts the `/flink-k8s-example` path on the node to create a volume.

The service account can be created from the default cluster service account:
{% highlight shell linenos %}
kubectl create serviceaccount flink-service-account

kubectl create clusterrolebinding flink-role-binding-flink --clusterrole=edit --serviceaccount=default:flink-service-account
{% endhighlight %}

We reference the service account in the flink cluster config:
{% highlight yaml linenos %}
apiVersion: flinkoperator.k8s.io/v1beta1
kind: FlinkCluster
metadata:
  name: beam-flink-cluster
  namespace: default
spec:
  flinkVersion: "1.15.2"
  image:
    name: apache/flink:1.15.2
  serviceAccountName: "flink-service-account"
  
  ...
{% endhighlight %}


The flink config needs to be updated to include the HA config:
{% highlight yaml linenos %}
taskmanager.memory.process.size: "2g"
taskmanager.data.port: "6121"
taskmanager.numberOfTaskSlots: "3"
parallelism.default: "1"

state.backend: filesystem
state.backend.incremental: "true"
state.checkpoints.dir: file:///flink-shared/checkpoints
state.savepoints.dir: file:///flink-shared/savepoints

classloader.resolve-order: parent-first

execution.checkpointing.interval: "60"

# Kubernetes config
kubernetes.cluster-id: "beam-flink-cluster"
kubernetes.taskmanager.service-account: "flink-service-account"

# Below for HA config
high-availability: org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory
high-availability.jobmanager.port: "50010"
high-availability.storageDir: file:///flink-shared/ha
high-availability.cluster-id: "beam-flink-cluster"
restart-strategy: fixed-delay
restart-strategy.fixed-delay.attempts: "10"
{% endhighlight %}

We are using the built in HA services factory class for high-availability. We mount the volume created previously as state storage and we set the cluster id to be the cluster name, which is required. 

Next we define the restart strategy and a timeout. As a precaution, I also pined the jobmanager port to 50010 as the configuration docs states that this port can be a random value when a new jobmanager is created. 

We define the state checkpoint interval to be 60. Note that this value must be greater than 0 for checkpointing to work. We define the cluster id, which is set to the name of the flink cluster. We also add the custom service account to the taskmanager.

The `state.backend` is set to filesystem. We also enable incremental checkpoint via `state.backend.incremental` which only stores diffs of checkpoints rather than entire checkpoints. Note that the `state.checkpoints.dir` and `state.savepoints.dir` can also be set to remote storage locations such as s3, but the main `high-availability.storageDir` has to be set to a volume.

Assuming the setup is right, when we start the flink session cluster, we should see the following in the jobmanager logs:
{% highlight shell linenos %}
2022-12-11 15:29:32,308 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesLeaderElector [] - Create KubernetesLeaderElector beam-flink-cluster-cluster-config-map with lock identity 3e81a3e8-b718-4c9e-96ad-cd8f0eacd48e.

2022-12-11 15:29:32,309 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Starting to watch for default/beam-flink-cluster-cluster-config-map, watching id:db45acf1-52b3-4239-91a6-7232c1c56bba
...

2022-12-11 15:29:32,378 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesLeaderElector [] - New leader elected 3e81a3e8-b718-4c9e-96ad-cd8f0eacd48e for beam-flink-cluster-cluster-config-map.

...

2022-12-11 15:29:32,541 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - Starting DefaultLeaderElectionService with org.apache.flink.runtime.leaderelection.MultipleComponentLeaderElectionDriverAdapter@8b670c0.

2022-12-11 15:29:32,541 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - Web frontend listening at http://beam-flink-cluster-jobmanager:8081.

2022-12-11 15:29:32,541 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - http://beam-flink-cluster-jobmanager:8081 was granted leadership with leaderSessionID=92a23e47-73ea-4564-890f-0cb39937a15a

...

2022-12-11 15:29:32,556 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - Starting DefaultLeaderElectionService with org.apache.flink.runtime.leaderelection.MultipleComponentLeaderElectionDriverAdapter@3513d214.

2022-12-11 15:29:32,556 INFO  org.apache.flink.runtime.resourcemanager.ResourceManagerServiceImpl [] - Starting resource manager service.

2022-12-11 15:29:32,556 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - Starting DefaultLeaderElectionService with org.apache.flink.runtime.leaderelection.MultipleComponentLeaderElectionDriverAdapter@7534785a.

2022-12-11 15:29:32,557 INFO  org.apache.flink.runtime.leaderretrieval.DefaultLeaderRetrievalService [] - Starting DefaultLeaderRetrievalService with KubernetesLeaderRetrievalDriver{configMapName='beam-flink-cluster-cluster-config-map'}.

2022-12-11 15:29:32,557 INFO  org.apache.flink.runtime.leaderretrieval.DefaultLeaderRetrievalService [] - Starting DefaultLeaderRetrievalService with KubernetesLeaderRetrievalDriver{configMapName='beam-flink-cluster-cluster-config-map'}.

2022-12-11 15:29:32,572 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Starting to watch for default/beam-flink-cluster-cluster-config-map, watching id:c389c018-0bda-437a-aea3-656aa64dc47f

2022-12-11 15:29:32,572 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Starting to watch for default/beam-flink-cluster-cluster-config-map, watching id:d25f089c-67eb-4211-8ca2-245e55409c37

2022-12-11 15:29:32,574 INFO  org.apache.flink.runtime.dispatcher.runner.DefaultDispatcherRunner [] - DefaultDispatcherRunner was granted leadership with leader id 92a23e47-73ea-4564-890f-0cb39937a15a. Creating new DispatcherLeaderProcess.

2022-12-11 15:29:32,581 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Start SessionDispatcherLeaderProcess.

2022-12-11 15:29:32,584 INFO  org.apache.flink.runtime.resourcemanager.ResourceManagerServiceImpl [] - Resource manager service is granted leadership with session id 92a23e47-73ea-4564-890f-0cb39937a15a.

2022-12-11 15:29:32,585 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Recover all persisted job graphs that are not finished, yet.

2022-12-11 15:29:32,607 INFO  org.apache.flink.runtime.rpc.akka.AkkaRpcService             [] - Starting RPC endpoint for org.apache.flink.runtime.resourcemanager.StandaloneResourceManager at akka://flink/user/rpc/resourcemanager_0 .

2022-12-11 15:29:32,619 INFO  org.apache.flink.runtime.jobmanager.DefaultJobGraphStore     [] - Retrieved job ids [] from KubernetesStateHandleStore{configMapName='beam-flink-cluster-cluster-config-map'}
2022-12-11 15:29:32,619 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Successfully recovered 0 persisted job graphs.
...
{% endhighlight %}

The logs show that the leader election service is activated and the current job manager http://beam-flink-cluster-jobmanager:8081 is selected to be the leader. Note that the service is created automatically via the flink-operator in this case. It then creates a dispatcher and resource manager service and assign them as leader, updating the configmaps.

HA automatically tracks the current leader via configmaps. It created two configmaps: `<flink cluster name>-cluster-config-map` and `<flink cluster name>-configmap`. The first configmap contains the leader election details while the second configmap contains a copy of the flink and log4j configs used in the initial cluster setup.

We can use the following failure scenarios to test if HA is actually working:

#### Kill the current jobmanager pod process

This will terminate the jobmanager process. The logs should show it being restarted:
{% highlight shell linenos %}
kubectl exec -it {jobmanager_pod_name} -- /bin/sh -c "kill 1"
{% endhighlight %}

Output of kubectl get pods:
{% highlight shell linenos %}
beam-flink-cluster-jobmanager-0    1/1     Running   1 (55s ago)   19m
beam-flink-cluster-taskmanager-0   2/2     Running   0             19m
beam-flink-cluster-taskmanager-1   2/2     Running   0             19m
{% endhighlight %}

The jobmanager logs show the same startup information as when the cluster was first created, suggesting that a new jobmanager pod was created.

#### Kill a taskmanager pod process

{% highlight shell linenos %}
kubectl exec -it {taskmanager_pod_name} -- /bin/sh -c "kill 1"
{% endhighlight %}

Output of the jobmanager logs shows that a new taskmanager process is started and registered:
{% highlight shell linenos %}
2022-12-11 15:51:47,574 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager [] - Closing TaskExecutor connection 10.244.0.18:6122-3821cd because: The TaskExecutor is shutting down.

2022-12-11 15:52:01,144 INFO  org.apache.flink.runtime.resourcemanager.StandaloneResourceManager [] - Registering TaskManager with ResourceID 10.244.0.18:6122-fe0c19 (akka.tcp://flink@10.244.0.18:6122/user/rpc/taskmanager_0) at ResourceManager
...
{% endhighlight %}

#### Delete the jobmanager pod

Delete the main jobmanager pod using:
{% highlight shell linenos %}
kubectl delete pod {jobmanager_pod_name}
{% endhighlight %}

Output of jobmanager logs:
{% highlight shell linenos %}
2022-12-11 15:55:28,061 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint        [] - RECEIVED SIGNAL 15: SIGTERM. Shutting down as requested.

2022-12-11 15:55:28,064 INFO  org.apache.flink.runtime.entrypoint.ClusterEntrypoint        [] - Shutting StandaloneSessionClusterEntrypoint down with application status UNKNOWN. Diagnostics Cluster entrypoint has been closed externally..

2022-12-11 15:55:28,064 INFO  org.apache.flink.runtime.blob.BlobServer                     [] - Stopped BLOB server at 0.0.0.0:6124

2022-12-11 15:55:28,068 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - Shutting down rest endpoint.

2022-12-11 15:55:28,078 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - Removing cache directory /tmp/flink-web-0b1e5be0-95c2-4716-9688-44ef1748278b/flink-web-ui

2022-12-11 15:55:28,080 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - Stopping DefaultLeaderElectionService.

2022-12-11 15:55:28,090 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - Shut down complete.

2022-12-11 15:55:28,091 INFO  org.apache.flink.runtime.entrypoint.component.DispatcherResourceManagerComponent [] - Closing components.

2022-12-11 15:55:28,091 INFO  org.apache.flink.runtime.leaderretrieval.DefaultLeaderRetrievalService [] - Stopping DefaultLeaderRetrievalService.

2022-12-11 15:55:28,091 INFO  org.apache.flink.kubernetes.highavailability.KubernetesLeaderRetrievalDriver [] - Stopping KubernetesLeaderRetrievalDriver{configMapName='beam-flink-cluster-cluster-config-map'}.

2022-12-11 15:55:28,091 INFO  org.apache.flink.runtime.leaderretrieval.DefaultLeaderRetrievalService [] - Stopping DefaultLeaderRetrievalService.

2022-12-11 15:55:28,091 INFO  org.apache.flink.kubernetes.highavailability.KubernetesLeaderRetrievalDriver [] - Stopping KubernetesLeaderRetrievalDriver{configMapName='beam-flink-cluster-cluster-config-map'}.

2022-12-11 15:55:28,091 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - Stopping DefaultLeaderElectionService.

2022-12-11 15:55:28,091 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Stopped to watch for default/beam-flink-cluster-cluster-config-map, watching id:dbf98f2b-6765-45be-bf62-afc1f15d84f6

2022-12-11 15:55:28,091 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Stopped to watch for default/beam-flink-cluster-cluster-config-map, watching id:a2cc3271-7ef2-47ee-9342-dfcbc83bbc4a

2022-12-11 15:55:28,110 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Stopping SessionDispatcherLeaderProcess.

2022-12-11 15:55:28,110 INFO  org.apache.flink.runtime.resourcemanager.ResourceManagerServiceImpl [] - Stopping resource manager service.

2022-12-11 15:55:28,110 INFO  org.apache.flink.runtime.dispatcher.StandaloneDispatcher     [] - Stopping dispatcher akka.tcp://flink@beam-flink-cluster-jobmanager:6123/user/rpc/dispatcher_1.

2022-12-11 15:55:28,111 INFO  org.apache.flink.runtime.leaderelection.DefaultLeaderElectionService [] - 
Stopping DefaultLeaderElectionService.

2022-12-11 15:55:28,111 INFO  org.apache.flink.runtime.dispatcher.StandaloneDispatcher     [] - Stopping all currently running jobs of dispatcher akka.tcp://flink@beam-flink-cluster-jobmanager:6123/user/rpc/dispatcher_1.

2022-12-11 15:55:28,112 INFO  org.apache.flink.runtime.dispatcher.StandaloneDispatcher     [] - Stopped dispatcher akka.tcp://flink@beam-flink-cluster-jobmanager:6123/user/rpc/dispatcher_1.

2022-12-11 15:55:28,114 INFO  org.apache.flink.runtime.jobmanager.DefaultJobGraphStore     [] - Stopping DefaultJobGraphStore.

2022-12-11 15:55:28,117 INFO  org.apache.flink.runtime.resourcemanager.slotmanager.DeclarativeSlotManager [] - Closing the slot manager.

2022-12-11 15:55:28,117 INFO  org.apache.flink.runtime.resourcemanager.slotmanager.DeclarativeSlotManager [] - Suspending the slot manager.

2022-12-11 15:55:28,119 INFO  org.apache.flink.runtime.leaderelection.DefaultMultipleComponentLeaderElectionService [] - Closing DefaultMultipleComponentLeaderElectionService.

2022-12-11 15:55:28,120 INFO  org.apache.flink.kubernetes.highavailability.KubernetesMultipleComponentLeaderElectionDriver [] - Closing org.apache.flink.kubernetes.highavailability.KubernetesMultipleComponentLeaderElectionDriver@5bb0c5c0.
{% endhighlight %}

The logs indicate that the leader election service was indeed closed when the jobmanager pod is deleted.

The output of the new jobmanager pod show that the new jobmanager pod is elected as the current leader:
{% highlight shell linenos %}
2022-12-11 15:55:36,536 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesLeaderElector [] - Create KubernetesLeaderElector beam-flink-cluster-cluster-config-map with lock identity 05df88b5-bbd2-400f-b1e8-9d4d386b2a43.

2022-12-11 15:55:36,537 INFO  org.apache.flink.kubernetes.kubeclient.resources.KubernetesConfigMapSharedInformer [] - Starting to watch for default/beam-flink-cluster-cluster-config-map, watching id:fa91664e-7df9-4937-802b-20e3cba09bc9

...

2022-12-11 15:55:46,149 INFO  org.apache.flink.runtime.dispatcher.DispatcherRestEndpoint   [] - http://beam-flink-cluster-jobmanager:8081 was granted leadership with leaderSessionID=28855791-dee8-4a20-9cd1-b1d5b91be838

2022-12-11 15:55:46,149 INFO  org.apache.flink.runtime.resourcemanager.ResourceManagerServiceImpl [] - Resource manager service is granted leadership with session id 28855791-dee8-4a20-9cd1-b1d5b91be838.
...
{% endhighlight %}

### Testing HA with checkpoints

We can use the built-in statemachine example to simulate a long running streaming job. Then we monitor the checkpoints directory to ensure that its created.

![Running statemachine streaming job](/assets/img/flink/running-streamjob.png)

The jobmanager logs should show the job running and checkpoints being saved:
{% highlight shell linenos %}
2022-12-11 16:02:21,632 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph       [] - Flat Map -> Sink: Print to Std. Out (1/1) (e4adc6c6d1d4b2b9e9318cfd95e65e35) switched from INITIALIZING to RUNNING.

2022-12-11 16:02:23,370 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 1 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1670774543316 for job c506b6c290cc2d96a0e3f0eea10395c4.

2022-12-11 16:02:23,451 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 1 for job c506b6c290cc2d96a0e3f0eea10395c4 (7735 bytes, checkpointDuration=123 ms, finalizationTime=11 ms).

2022-12-11 16:02:25,326 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 2 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1670774545316 for job c506b6c290cc2d96a0e3f0eea10395c4.

2022-12-11 16:02:25,364 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 2 for job c506b6c290cc2d96a0e3f0eea10395c4 (8320 bytes, checkpointDuration=23 ms, finalizationTime=25 ms).

...
{% endhighlight %}

We can try to kill the jobmanager process and it should resume the job from the last checkpoint, which was checkpoint 106 for this example:

{% highlight shell linenos %}
2022-12-11 16:06:08,109 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Recover all persisted job graphs that are not finished, yet.

2022-12-11 16:06:08,187 INFO  org.apache.flink.runtime.jobmanager.DefaultJobGraphStore     [] - Retrieved job ids [c506b6c290cc2d96a0e3f0eea10395c4] from KubernetesStateHandleStore{configMapName='beam-flink-cluster-cluster-config-map'}

2022-12-11 16:06:08,188 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Trying to recover job with job id c506b6c290cc2d96a0e3f0eea10395c4.

2022-12-11 16:06:08,406 INFO  org.apache.flink.runtime.jobmanager.DefaultJobGraphStore     [] - Recovered JobGraph(jobId: c506b6c290cc2d96a0e3f0eea10395c4).

2022-12-11 16:06:08,407 INFO  org.apache.flink.runtime.dispatcher.runner.SessionDispatcherLeaderProcess [] - Successfully recovered 1 persisted job graphs.

2022-12-11 16:06:09,128 INFO  org.apache.flink.runtime.jobmaster.JobMaster                 [] - Initializing job 'State machine job' (c506b6c290cc2d96a0e3f0eea10395c4).

2022-12-11 16:06:09,209 INFO  org.apache.flink.runtime.jobmaster.JobMaster                 [] - Using restart back off time strategy FixedDelayRestartBackoffTimeStrategy(maxNumberRestartAttempts=10, backoffTimeMS=1000) for State machine job (c506b6c290cc2d96a0e3f0eea10395c4).

2022-12-11 16:06:09,230 INFO  org.apache.flink.runtime.checkpoint.DefaultCompletedCheckpointStoreUtils [] - Recovering checkpoints from KubernetesStateHandleStore{configMapName='beam-flink-cluster-c506b6c290cc2d96a0e3f0eea10395c4-config-map'}.

2022-12-11 16:06:09,276 INFO  org.apache.flink.runtime.checkpoint.DefaultCompletedCheckpointStoreUtils [] - Found 1 checkpoints in KubernetesStateHandleStore{configMapName='beam-flink-cluster-c506b6c290cc2d96a0e3f0eea10395c4-config-map'}.

2022-12-11 16:06:09,277 INFO  org.apache.flink.runtime.checkpoint.DefaultCompletedCheckpointStoreUtils [] - Trying to fetch 1 checkpoints from storage.

2022-12-11 16:06:09,277 INFO  org.apache.flink.runtime.checkpoint.DefaultCompletedCheckpointStoreUtils [] - Trying to retrieve checkpoint 106.
...

2022-12-11 16:06:09,504 INFO  org.apache.flink.runtime.jobmaster.JobMaster                 [] - Starting execution of job 'State machine job' (c506b6c290cc2d96a0e3f0eea10395c4) under job master id 8c15f93c28f74b20cc1b30ccb4da4cc3.

2022-12-11 16:06:09,506 INFO  org.apache.flink.runtime.jobmaster.JobMaster                 [] - Starting scheduling with scheduling strategy [org.apache.flink.runtime.scheduler.strategy.PipelinedRegionSchedulingStrategy]

2022-12-11 16:06:09,507 INFO  org.apache.flink.runtime.executiongraph.ExecutionGraph       [] - Job State machine job (c506b6c290cc2d96a0e3f0eea10395c4) switched from state CREATED to RUNNING.
...

2022-12-11 16:06:18,799 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 107 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1670774778787 for job c506b6c290cc2d96a0e3f0eea10395c4.

2022-12-11 16:06:18,884 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 107 for job c506b6c290cc2d96a0e3f0eea10395c4 (8338 bytes, checkpointDuration=72 ms, finalizationTime=25 ms).

2022-12-11 16:06:20,803 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Triggering checkpoint 108 (type=CheckpointType{name='Checkpoint', sharingFilesStrategy=FORWARD_BACKWARD}) @ 1670774780787 for job c506b6c290cc2d96a0e3f0eea10395c4.

2022-12-11 16:06:20,843 INFO  org.apache.flink.runtime.checkpoint.CheckpointCoordinator    [] - Completed checkpoint 108 for job c506b6c290cc2d96a0e3f0eea10395c4 (15169 bytes, checkpointDuration=32 ms, finalizationTime=24 ms).
...
{% endhighlight %}

As can be seen above, the job was restored and continued to create checkpoints from its last checkpoint.

When the job is stopped/cancelled manually, the HA data, including the checkpoints will also be automatically deleted:
{% highlight shell linenos %}
...

2022-12-11 16:16:34,782 INFO  org.apache.flink.kubernetes.highavailability.KubernetesMultipleComponentLeaderElectionHaServices [] - Clean up the high availability data for job 034b3fe2d4f673aed68b38e787f8edf0.

2022-12-11 16:16:34,788 INFO  org.apache.flink.kubernetes.highavailability.KubernetesMultipleComponentLeaderElectionHaServices [] - Finished cleaning up the high availability data for job 034b3fe2d4f673aed68b38e787f8edf0.

2022-12-11 16:16:34,797 INFO  org.apache.flink.runtime.jobmanager.DefaultJobGraphStore     [] - Removed job graph 034b3fe2d4f673aed68b38e787f8edf0 from KubernetesStateHandleStore{configMapName='beam-flink-cluster-cluster-config-map'}.
{% endhighlight %}

This is because checkpoints, by default, are only used to resume a job from failures. However you can set the application configuration to retain the checkpoint on job cancellation.

Further information can be found in the [Documentation on Flink HA]. Details about Flink configuration can be found on [Flink 1.15.2 configuration] page. More information can also be found on [Flink Checkpoints] and [Checkpoints vs savepoints].

### Summary

This post attempts to explain how to setup high availability for a flink cluster running locally in a kind cluster but the same can be applied in an actual deployed flink cluster. In that case, the state backend should be changed to `rocksdb` as well as other tweaks to the configuration which is for another article.

H4ppy H4ck1n6 !!! 