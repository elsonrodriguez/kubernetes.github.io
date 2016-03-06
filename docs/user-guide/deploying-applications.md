---
---

You previously read about how to quickly deploy a simple replicated application using [`kubectl run`](/docs/user-guide/quick-start) and how to configure and launch single-run containers using pods ([Configuring containers](/docs/user-guide/configuring-containers)). Here you'll use the configuration-based approach to deploy a continuously running, replicated application.

* TOC
{:toc}

## Launching a set of replicas using a configuration file

Kubernetes creates and manages sets of replicated containers (actually, replicated [Pods](/docs/user-guide/pods)) using [*Replication Controllers*](/docs/user-guide/replication-controller).

A replication controller simply ensures that a specified number of pod "replicas" are running at any one time. If there are too many, it will kill some. If there are too few, it will start more. It's analogous to Google Compute Engine's [Instance Group Manager](https://cloud.google.com/compute/docs/instance-groups/manager/) or AWS's [Auto-scaling Group](http://docs.aws.amazon.com/AutoScaling/latest/DeveloperGuide/AutoScalingGroup) (with no scaling policies).

The replication controller created to run nginx by `kubectl run` in the [Quick start](/docs/user-guide/quick-start) could be specified using YAML as follows:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Some differences compared to specifying just a pod are that the `kind` is `ReplicationController`, the number of `replicas` desired is specified, and the pod specification is under the `template` field. The names of the pods don't need to be specified explicitly because they are generated from the name of the replication controller.
View the [replication controller API
object](http://kubernetes.io/v1.1/docs/api-reference/v1/definitions/#_v1_replicationcontroller)
to view the list of supported fields.

This replication controller can be created using `create`, just as with pods:

```shell
$ kubectl create -f ./nginx-rc.yaml
replicationcontrollers/my-nginx
```

Unlike in the case where you directly create pods, a replication controller replaces pods that are deleted or terminated for any reason, such as in the case of node failure. For this reason, we recommend that you use a replication controller for a continuously running application even if your application requires only a single pod, in which case you can omit `replicas` and it will default to a single replica.

## Viewing replication controller status

You can view the replication controller you created using `get`:

```shell
$ kubectl get rc
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR    REPLICAS
my-nginx     nginx          nginx      app=nginx   2
```

This tells you that your controller will ensure that you have two nginx replicas.

You can see those replicas using `get`, just as with pods you created directly:

```shell
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-065jq   1/1       Running   0          51s
my-nginx-buaiq   1/1       Running   0          51s
```

## Deleting replication controllers

When you want to kill your application, delete your replication controller, as in the [Quick start](/docs/user-guide/quick-start):

```shell
$ kubectl delete rc my-nginx
replicationcontrollers/my-nginx
```

By default, this will also cause the pods managed by the replication controller to be deleted. If there were a large number of pods, this may take a while to complete. If you want to leave the pods running, specify `--cascade=false`.

If you try to delete the pods before deleting the replication controller, it will just replace them, as it is supposed to do.

## Labels

Kubernetes uses user-defined key-value attributes called [*labels*](/docs/user-guide/labels) to categorize and identify sets of resources, such as pods and replication controllers. The example above specified a single label in the pod template, with key `app` and value `nginx`. All pods created carry that label, which can be viewed using `-L`:

```shell
$ kubectl get pods -L app
NAME             READY     STATUS    RESTARTS   AGE       APP
my-nginx-afv12   0/1       Running   0          3s        nginx
my-nginx-lg99z   0/1       Running   0          3s        nginx
```

The labels from the pod template are copied to the replication controller's labels by default, as well -- all resources in Kubernetes support labels:

```shell
$ kubectl get rc my-nginx -L app
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR    REPLICAS   APP
my-nginx     nginx          nginx      app=nginx   2          nginx
```

More importantly, the pod template's labels are used to create a [`selector`](/docs/user-guide/labels/#label-selectors) that will match pods carrying those labels. You can see this field by requesting it using the [Go template output format of `kubectl get`](/docs/user-guide/kubectl/kubectl_get):

```shell
$ kubectl get rc my-nginx -o template --template="{{.spec.selector}}"
map[app:nginx]
```

You could also specify the `selector` explicitly, such as if you wanted to specify labels in the pod template that you didn't want to select on, but you should ensure that the selector will match the labels of the pods created from the pod template, and that it won't match pods created by other replication controllers. The most straightforward way to ensure the latter is to create a unique label value for the replication controller, and to specify it in both the pod template's labels and in the selector.

## What's next?

[Learn about exposing applications to users and clients, and connecting tiers of your application together.](/docs/user-guide/connecting-applications)