# Red Hat - PV Pool Operator Assignment

## Intro
A common pattern in kubernetes is the [operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/). An operator is a piece of software, that is incharge of managing all the different components of an application deployment in a kubernetes cluster.  
The interface between the user and the operator is usually done using **custom resource definitions** ([CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)). These are resources that are not part of the core kubernetes code which are defined by the operator maintainer.  
Operators work by reading the CRD which describes how the user wants to deploy the app. The operator then starts deploying different componenets until it gets to the desired state. This is done iteratively in a reconcile loop that is triggered as long as there are events that the operator is watching.

## PV Pool Operator

In this task, you will work on an operator called pv-pool-operator. This operator is managing a pool of [pods](https://kubernetes.io/docs/concepts/workloads/pods/), each running a storage agent that can accept reads and writes from\to a persistent volume([PV](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)). In this exercise the pods are running a simulated agent that is "connected" to a central storage system which uses the pool as a storage backing store.  
The PV pool is defined using a `PvPool` CRD. here is an example CR defining a pool:
```
apiVersion: pvpool.noobaa.com/v1
kind: PvPool
metadata:
  name: pvpool-1
spec:
  image: storage-agent:1
  numPVs: 3
  pvSizeGB: 5
```
The CRD defines the image for the storage agent, the number of agents and the PV size for each agent


### The Project 
* You can download the project from [here](https://drive.google.com/file/d/1DxXeGRAqPtzFCJsfRd08r30iXOVarWiZ/view?usp=sharing)

* This project is in golang and is using the Kubernetes API. It is recommended to get yourself familiarized with golang and basic kubernetes concepts and commands. The [official Kubernetes docs](https://kubernetes.io/docs/home/) are pretty good. 

* The pv-pool-operator was created using [Operator-SDK for golang](https://sdk.operatorframework.io/docs/building-operators/golang/quickstart/). Try reading a little bit and understand how operators are built using operator-sdk, it will help you to better understand the pv-pool-operator code.

* The main reconcile function can be found in `controllers/pvpool_controller.go`. 

* Requirements to build and deploy
  * go 1.13+
  * [operator-sdk](https://sdk.operatorframework.io/docs/building-operators/golang/installation/) and its requirements (tested with v1.3.0)
  * [minikube](https://minikube.sigs.k8s.io/docs/start/) - a development, single node, kubernetes cluster that runs locally on your machine
  * [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
  * docker

### Building and Deploying
* All build and deploy commands are defined in the `Makefile`. Feel free to add\edit any commands in the file
* Deploying the operator and the storage-agents in minikube requires a docker image to be pushed to a container registry such as hub.docker.com or quay.io, so you might want to sign up to one of these services
* Alternatively, you can use the internal docker daemon inside minikube to build your docker images. In that case, you will not need to push it to an external registry since it is already present in the cluster. you can define docker to work on the minikube docker using the command
   ```
   eval $(minikube docker-env)
   ```
   of course you will need to start minikube first with `minikube start`
* Once you have everything set up you can start building the different images
    * Build storage-agent image. This will build an image named `storage-agent:1`. You can edit the `Makefile` to change it.
        ```
        make docker-build-storage-agent
        ```
    * Build the operator and tag the image in the format `your-docker-account/pv-pool-repo-name:tag`. e.g:
       ```
       make docker-build IMG=noobaa/pv-pool-operator:1
       ```
    * If you are not working locally (in minikube docker) you will need to push your images to a registry:
       ```
       docker push [storage-agent image]
       docker push [pv-pool operator image]
       ```
    * Deploy the operator - this will run an operator [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) in your cluster
       ```
       make deploy IMG=[your operator image]
       ```
       Once deployed you will see the operator pod running in the pv-pool-operator-system namespace
       ```
       ❱ kubectl get pod -n pv-pool-operator-system

       NAME                                                   READY   STATUS    RESTARTS   AGE
       pv-pool-operator-controller-manager-7c58dc6868-2klzq   2/2     Running   0          12s
       ```

    * Once the operator is in `Ready` state you can create a `PvPool` custom resource (CR) and the operator should start and deploy the PV pool. An example for a pv pool CR is in `config/samples/pvpool_v1_pvpool.yaml`. you can edit the Spec and apply it to the cluster:
       ```
       ❱ kubectl apply -f config/samples/pvpool_v1_pvpool.yaml
       pvpool.pvpool.noobaa.com/pvpool-1 created
       ```

    * It will take some time, but eventually you should see these resources deployd in the cluster:
        ```
        ❱ kubectl -n pv-pool-operator-system get all
        NAME                                                       READY   STATUS    RESTARTS   AGE
        pod/pv-pool-operator-controller-manager-7c58dc6868-2klzq   2/2     Running   0          10m
        pod/pvpool-1-sts-0                                         1/1     Running   0          28s
        pod/pvpool-1-sts-1                                         1/1     Running   0          22s
        pod/pvpool-1-sts-2                                         1/1     Running   0          14s

        NAME                                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
        service/pv-pool-operator-controller-manager-metrics-service   ClusterIP   10.96.56.243     <none>        8443/TCP   10m
        service/pvpool-1-srv                                          ClusterIP   10.104.249.218   <none>        8080/TCP   28s

        NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/pv-pool-operator-controller-manager   1/1     1            1           10m

        NAME                                                             DESIRED   CURRENT   READY   AGE
        replicaset.apps/pv-pool-operator-controller-manager-7c58dc6868   1         1         1       10m

        NAME                            READY   AGE
        statefulset.apps/pvpool-1-sts   3/3     28s
        ```
    * you can follow the operator logs with 
       ```
       kubectl log -f -c manager [pv-pool pod name]
       ```
       in general you can see the logs of any pod with `kubectl log [pod name]`. Some pods (like the operator) have more than one container, so you need to specify which container you want.

### The operator 
When a creation or update of a PV pool CR is recognized, the pv-pool operator reconcile loop is triggered.

The operator deploys these resources for each pv-pool
* [Service](https://kubernetes.io/docs/concepts/services-networking/service/) - A `ClusterIP` service to expose the storage agents pods
* [Statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) - Manages the storage-agents pods. each pod is mounted with a [PV](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (using a [PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)).)

On each run of the reconcile loop the operator is updating the `Status` of the pv-pool CR (you can look at the CR description with `kubectl get pvpool pvpool-1 -o yaml`)

Once the pv pool is reconciled the Status is set to `Ready`

### The storage-agents
The storage agents that run inside each pod are simulating an agent that is connected to the main storage system and receiving IO operations. The storage agents have a simple rest API to get their status and to perform decommission (withdraw from the service). You have helper functions in the code to call these API requests

## The Task
1. Your first task is to add the used storage percentage of the pool to the `PvPoolStatus` (defined in `api/v1/pvpool_types.go`). Each storage agent is reporting its total and used storage in the `status` API request.
2. Your second task is to better handle the scaling of the pool (increasing or decreasing `numPVs`).  
    * In the current implementation, when you update `numPVs` in the pvpool CR (using `kubectl edit pvpool [CR name]` or applying the yaml with new values), the operator will update the statefulset number of replicas. When scaling down this will remove the excess pods without notice, and in a real system can cause data loss.  
    * You should change this behavior by decommissioning each pod (use `decommissionStorageAgent` function in the code) before it is removed. After decommission is called the storage-agent will enter a `decommissioning` state. Once the decommissioning process is completed the storage-agent will transition to the `decommissioned` state, only then it is safe to remove the pods (the storage agent states are defined as `PvPodStatus` in `api/v1/pvpool_types.go`).  
    * Notice that due to the way [statefulsets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) work, you cannot scale up while there are pods that are in `decommissioning` or `decommissioned` state.

You should upload the code to a private github repo. It's recommended to do that before you start working and use git for version control.

Good luck!!
