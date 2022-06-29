# Redis Keda

This project is a proof of concept for event-driven architecture on Kubernetes utilising Keda + Redis.

Inspired
by [KEDA: Event Driven and Serverless Containers in Kubernetes - Jeff Hollan, Microsoft](https://www.youtube.com/watch?v=ZK2SS_GXF-g&t=1631s)

## Requirements

- Kubernetes
- Keda

### Install keda using helm

```bash
helm repo add kedacore https://kedacore.github.io/charts
kelm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

## Project Structure

### Redis/

[Redis](https://redis.io/) is a light weight in-memory data structure store that can be used as a database or messaging
queue. This project leverages Redis lists as a message queue and scaling metric for our consumer pods.

### Publisher/

The publisher/ directory contains a simple Python script that periodically appends batches of events to a list in Redis.
The format of the events being published are as follows:

```json5
{
  "wait": 17
} // random between 10 and 20
```

This script is packaged into a dockerfile that is published
to [scottzach1/redis-publisher](https://hub.docker.com/repository/docker/scottzach1/redis-publisher).

### Consumer/

The consumer directory contains a simple Python script that reads the first event in a Redis list. The script will then
parse the event and sleep for `wait` number of seconds before terminating.

Although extremely primitive, this provides a simple variable "IO blocking" workload that we can attempt to scale using
Keda [accurate scaling](https://keda.sh/docs/2.7/concepts/scaling-jobs/).

This script is packaged into a dockerfile that is published
to [scottzach1/redis-consumer](https://hub.docker.com/repository/docker/scottzach1/redis-consumer).

#### Keda

[Keda (Kubernetes Event-driven Autoscaling)](https://keda.sh/) is an event based autoscaler that can be leveraged to
drive scaling in Kubernetes in useful ways based off a number of strategies.

This project utilises the [Redis Lists](https://keda.sh/docs/2.7/scalers/redis-streams/) trigger to scale the number of
consumer pods based on the number of respective events in the Redis list. Although Kubernetes provides native scaling
strategies with
[HPA (Horizontal Pod Autoscaling)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), this is
typically limited to resource metrics such as memory or CPU usage.

Unfortunately, this isn't always ideal in scenarios such as highly IO bound workloads. In such case, Kubernetes HPA
could retire a node before it had finished a lengthily IO bound job. Instead, by using Keda you can enforce various
workflows such as a 1 pod per 1 event lifecycle that is scaled automagically via a
Keda [ScaledJob](https://keda.sh/docs/2.7/concepts/scaling-jobs/). Here, we are also using `accurate` scaling method as
each job execution will consume an event from the queue.

## Setup Commands

First we will create a namespace for our project.

```bash
kubectl create namespace redis-demo
```

### Redis

```bash
kubectl -n redis-demo apply -f manifests/redis-deployment.yaml

# To see created resources:
kubectl -n redis-demo get all
#----------------------------
NAME                         READY   STATUS    RESTARTS   AGE
pod/redis-8476db56bf-mnzpg   1/1     Running   0          57s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/redis   ClusterIP   10.98.200.15   <none>        6379/TCP   57s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/redis   1/1     1            1           57s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/redis-8476db56bf   1         1         1       57s
```

### Publisher

```bash
kubectl -n redis-demo apply -f manifests/publisher-deployment.yaml

# To see created resources:
kubectl -n redis-demo get all
#----------------------------
NAME                                   READY   STATUS    RESTARTS   AGE
pod/redis-8476db56bf-mnzpg             1/1     Running   0          2m38s
pod/redis-publisher-6c45f5c474-g9gtq   1/1     Running   0          6s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/redis   ClusterIP   10.98.200.15   <none>        6379/TCP   2m38s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/redis             1/1     1            1           2m38s
deployment.apps/redis-publisher   1/1     1            1           6s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/redis-8476db56bf             1         1         1       2m38s
replicaset.apps/redis-publisher-6c45f5c474   1         1         1       6s
```

We can then inspect the container output:

```bash
kubectl -n redis-demo logs redis-publisher-6c45f5c474-g9gtq
publishing events:
- {'wait': 15}
- {'wait': 17}
- {'wait': 15}
- {'wait': 13}
- {'wait': 10}
- {'wait': 17}
- {'wait': 20}
- {'wait': 11}
- {'wait': 10}
- {'wait': 14}

sleeping for 10s
```

As well as inspect the redis list:

```bash
kubectl -n redis-demo exec -it redis-8476db56bf-mnzpg -- redis-cli LLEN events
#----------------------------
(integer) 10  # this will increase w/ time
```

### Consumer

**TODO: MAKE SURE TO UPDATE THE NAMESPACE IN REDIS HOST IN [consumer-scaledjob.yaml](./manifests/consumer-scaledjob.yaml)**

```bash
kubectl -n redis-demo apply -f manifests/consumer-scaledjob.yaml

# To see the scaled job:
kubectl -n redis-demo get scaledjobs/redis-scaledjob
```

You should now see the number of jobs start to increase:

```bash
watch kubectl -n redis-demo get all
#---------------------------------- 
Every 2.0s: minikube kubectl -- -n redis-demo get all                                Arch-3080: Fri May 27 00:00:03 2022
NAME                          READY   STATUS   RESTARTS   AGE      RESTARTS   AGE
pod/redis-8476db56bf-bzbgq    1/1     Running  0          20m      0          21m
pod/redis-publisher-6c45f5c474-4b5mx   1/1     Running rCreating   0          26s
pod/redis-scaledjob-4md4k-jc7gs        0/1     ContainerCreating   0          22s
pod/redis-scaledjob-4tz99-csssc        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-6nqhc-wtjmm        0/1     ContainerCreating   0          22s
pod/redis-scaledjob-7s6kh-6msq4        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-9p289-7bwql        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-g72g7-jtgh9        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-jg67k-86zs5        0/1     ContainerCreating   0          1s
pod/redis-scaledjob-jl8fp-tdhm9        0/1     ContainerCreating   0          1s
pod/redis-scaledjob-jmjjf-d8r9j        0/1     ContainerCreating   0          22s
pod/redis-scaledjob-kmlj9-m87rj        0/1     ContainerCreating   0          1s
pod/redis-scaledjob-lb85d-l6phw        0/1     ContainerCreating   0          22s
pod/redis-scaledjob-mlpfs-bpjlj        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-mw29g-f56rg        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-mwrrl-gvxxd        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-n674s-76pnn        0/1     ContainerCreating   0          1s
pod/redis-scaledjob-qhj6z-wdw8l        0/1     ContainerCreating   0          1s
pod/redis-scaledjob-tpr8l-2tk6k        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-xcp2c-b7lz8        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-xpnnx-rk5vn        0/1     ContainerCreating   0          10s
pod/redis-scaledjob-xwldv-s52kh        0/1     ContainerCreating   0          22s
...
```

You may also watch the number of events:

```bash
watch kubectl -n redis exec -it redis-demo redis-8476db56bf-bzbgq -- redis-cli LLEN events
#----------------------------------
Every 2.0s: minikube kubectl -- -n redis-demo exec -it redis-8476db56bf-bzbgq -- redis-cli LLEN events        Arch-3080: Fri May 27 00:36:04 2022

(integer) 7
```
