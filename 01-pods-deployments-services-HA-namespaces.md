# Pods and Deployments:

A **Pod** (*not container*) is the basic building-block/worker-unit in Kubernetes. Normally a pod is a part of a **Deployment**. 


## Creating pods using 'run' command:
We start by creating our first deployment. Normally people will run an nginx container/pod as first example o deployment. You can surely do that. But, we will run a different container image as our first exercise. The reason is that it will work as a multitool for testing and debugging throughout this course. Besides it too runs nginx! 


Here is the command to do it:

```
kubectl run multitool  --image=praqma/network-multitool
```

You should be able to see the following output:

```
$ kubectl run multitool --image=praqma/network-multitool 
deployment "multitool" created
```

What this command does behind the scenes is, it creates a deployment named multitool, starts a pod using this docker image (praqma/network-multitool), and makes that pod a member of that deployment. You don't need to confuse yourself with all these details at this stage. This is just extra (but vital) information. Just so you know what we are talking about, check the list of pods and deployments:

List of pods:
```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          3m
```

List of deployments:
```
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool   1         1         1            1           3m
```

There is actually also a replicaset, which is created as a result of the `run` command above, but we did not mention earlier; because that is not super important to know at this point. It is something which deals with the number of copies of this pod. It will be covered in later exercise. It is shown below just for the sake of completeness.
```
$ kubectl get replicasets
NAME                   DESIRED   CURRENT   READY     AGE
multitool-3148954972   1         1         1         3m
```

Ok. The bottom line is that we wanted to have a pod running ,and we have that. 


Lets setup another pod, a traditional nginx deployment, with a specific version - 1.7.9. 


Setup an nginx deployment with nginx:1.7.9
```
kubectl run nginx  --image=nginx:1.7.9
```

You get another deployment and a replicaset as a result of above command, shown below, so you know what to expect:

```
$ kubectl get pods,deployments,replicasets
NAME                            READY     STATUS    RESTARTS   AGE
po/multitool-3148954972-k8q06   1/1       Running   0          25m
po/nginx-1480123054-xn5p8       1/1       Running   0          14s

NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/multitool   1         1         1            1           25m
deploy/nginx       1         1         1            1           14s

NAME                      DESIRED   CURRENT   READY     AGE
rs/multitool-3148954972   1         1         1         25m
rs/nginx-1480123054       1         1         1         14s
```


## Alternate / preferred way to deploy pods:
You can also use the `nginx-simple-deployment.yaml` file to create the same nginx deployment. The file is in support-files directory of this repo. However before you execute the command shown below, note that it will try to create a deployment with the name **nginx**. If you already have a deployment named **nginx** running, as done in the previous step, then you will need to delete that first.

Delete the existing deployment using the following command:
```
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool   1         1         1            1           32m
nginx       1         1         1            1           7m

$ kubectl delete deployment nginx
deployment "nginx" deleted
```

Now you are ready to proceed with the example below:

```
$ kubectl create -f nginx-simple-deployment.yaml 
deployment "nginx" created
```


The contents of `nginx-simple-deployment.yaml` are as follows:
```
$ cat nginx-simple-deployment.yaml

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```


Verify that the deployment is created:
```
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool   1         1         1            1           59m
nginx       1         1         1            1           36s
```


Check if the pods are running:
```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          1h
nginx-431080787-9r0lx        1/1       Running   0          40s
```

## Deleting a pod, the Kubernetes promise of resilience:

Before we move forward, lets see if we can delete a pod, and if it comes to life automatically:
```
$ kubectl delete pod nginx-431080787-9r0lx 
pod "nginx-431080787-9r0lx" deleted
```

As soon as we delete a pod, a new one is created, satisfying the desired state by the deployment, which is - it needs at least one pod running nginx. So we see that a **new** nginx pod is created (with a new ID):
```
$ kubectl get pods
NAME                         READY     STATUS              RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running             0          1h
nginx-431080787-tx5m7        0/1       ContainerCreating   0          5s

(after few more seconds)

$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          1h
nginx-431080787-tx5m7        1/1       Running   0          12s
```

## Accessing the pods:
Now the question comes, How can we access nginx webserver at port 80 in this pod? For now we can do it from within the cluster. First, we need to know the IP address of the nginx pod. We use the `-o wide` parameters with the `get pods` command:

```
$ kubectl get pods -o wide
NAME                         READY     STATUS    RESTARTS   AGE       IP             NODE
multitool-3148954972-k8q06   1/1       Running   0          1h        100.96.2.31    ip-172-20-60-255.eu-central-1.compute.internal
nginx-431080787-tx5m7        1/1       Running   0          12m       100.96.1.148   ip-172-20-49-54.eu-central-1.compute.internal
```

**Bonus info:** The IPs you see for the pods (e.g. 100.96.2.31) are private IPs and belong to something called *Pod Network*, which is a completely private network inside a Kubernetes cluster, and is not accessible from outside the Kubernetes cluster.

Now, we `exec` into our multitool, as shown below and use the `curl` command from the pod to access nginx service in the nginx pod:

```
$ kubectl exec -it multitool-3148954972-k8q06 bash
[root@multitool-3148954972-k8q06 /]#
```

```
[root@multitool-3148954972-k8q06 /]# curl -s 100.96.1.148 | grep h1
<h1>Welcome to nginx!</h1>
[root@multitool-3148954972-k8q06 /]# 
```

We accessed the nginx service in the nginx pod using another pod in the same setup, because at this point in time the nginx web-service (running as pod) is not accessible using any **access methods**. These are explained next.

## Accessing a service:

To access any service inside any given pod (e.g. nginx web service), we need to *expose* the related deployment as a *service*. We have three main ways of exposing the deployment , or in other words, we have three ways to define a *service* , which we can access in three different ways. A service is (normally) created on top of an existing deployment.

### Service type: ClusterIP
Expose the deployment as a service - type=ClusterIP:
```
$ kubectl expose deployment nginx --port 80 --type ClusterIP
service "nginx" exposed
```

Check the list of services:
```
$ kubectl get services
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   100.64.0.1       <none>        443/TCP   2d
nginx        ClusterIP   100.70.204.237   <none>        80/TCP    4s
```

Notice, there are two services listed here. The first one is named **kubernetes**, which is the default service created (automatically) when a kubernetes cluster is created for the first time. It does not have any EXTERNAL-IP. This service is not our focus right now.

The service in focus is nginx, which does not have any external IP, nor does it say anything about any other ports except 80/TCP. This means it is not accessible over internet, but we can still access it from within cluster , using the service IP, not the pod IP. Lets see if we can access this service from our multitool.

```
[root@multitool-3148954972-k8q06 /]# curl -s 100.70.204.237 | grep h1
<h1>Welcome to nginx!</h1>
[root@multitool-3148954972-k8q06 /]# 
```

It worked! 

You can also use the `describe` command to describe any Kubernetes object in more detail. e.g. we use `describe` to see more details about our nginx service:

```
$ kubectl describe service nginx
Name:              nginx
Namespace:         default
Labels:            app=nginx
Annotations:       <none>
Selector:          app=nginx
Type:              ClusterIP
IP:                100.70.204.237
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         100.96.1.148:80
Session Affinity:  None
Events:            <none>
```

You can of-course use `... describe pod ...` , ` ... describe deployment ...` , etc.


**Additional notes about the Cluster-IP:**
* The IPs assigned to services as Cluster-IP are from a different Kubernetes network called *Service Network*, which is a completely different network altogether. i.e. it is not connected (nor related) to pod-network or the infrastructure network. Technically it is actually not a real network per-se; it is a labeling system, which is used by Kube-proxy on each node to setup correct iptables rules. (This is an advanced topic, and not our focus right now).
* No matter what type of service you choose while *exposing* your deployment, Cluster-IP is always assigned to that particular service.
* Every service has end-points, which point to the actual pods serving as a backends of a particular service.
* As soon as a service is created, and is assigned a Cluster-IP, an entry is made in Kubernetes' internal DNS against that service, with this service name and the Cluster-IP. e.g. `nginx.default.svc.cluster.local` would point to `100.70.204.237` . 


### Service type: NodePort

Our nginx service is still not reachable from outside, so now we re-create this service as NodePort.

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   100.64.0.1       <none>        443/TCP   17h
nginx        ClusterIP   100.70.204.237   <none>        80/TCP    15m
```

```
$ kubectl delete svc nginx
service "nginx" deleted
```

```
$ kubectl expose deployment nginx --port 80 --type NodePort
service "nginx" exposed
```

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   100.64.0.1     <none>        443/TCP        17h
nginx        NodePort    100.65.29.172  <none>        80:32593/TCP   8s
```

Notice that we still don't have an external IP, but we now have an extra port `32593` for this pod. This port is a **NodePort** exposed on the worker nodes. So now, if we know the IP of our nodes, we can access this nginx service from the internet. First, we find the public IP of the nodes:
```
$ kubectl get nodes -o wide
NAME                                            STATUS    ROLES     AGE       VERSION        EXTERNAL-IP     OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-dcn-cluster-35-default-pool-dacbcf6d-3918   Ready     <none>    17h       v1.8.8-gke.0   35.205.22.139   Container-Optimized OS from Google   4.4.111+         docker://17.3.2
gke-dcn-cluster-35-default-pool-dacbcf6d-c87z   Ready     <none>    17h       v1.8.8-gke.0   35.187.90.36    Container-Optimized OS from Google   4.4.111+         docker://17.3.2
```

Even though we have only one pod (and two worker nodes), we can access any of the node with this port, and it will eventually be routed to our pod. So, lets try to access it from our local work computer:

```
$ curl -s 35.205.22.139:32593 | grep h1
<h1>Welcome to nginx!</h1>
```

It works!

### Service type: LoadBalancer
So far so good; but, we do not expect the users to know the IP addresses of our worker nodes. It is not a flexible way of doing things. So we re-create the service as `type=LoadBalancer`. The type LoadBalancer is only available for use, if your k8s cluster is setup in any of the public cloud providers, GCE, AWS, etc.

```
[demo@kworkhorse exercises]$ kubectl delete svc nginx
service "nginx" deleted
```

```
$ kubectl expose deployment nginx --port 80 --type LoadBalancer
service "nginx" exposed
```

```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      100.64.0.1     <none>        443/TCP        17h
nginx        LoadBalancer   100.69.15.89   <pending>     80:31354/TCP   5s
```

In few minutes of time the external IP will have some value instead of the word 'pending' . 
```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      100.64.0.1     <none>        443/TCP        17h
nginx        LoadBalancer   100.69.15.89   35.205.60.29  80:31354/TCP   5s

```

Now, we can access this service without using any special port numbers:
```
[demo@kworkhorse exercises]$ curl -s 35.205.60.29 | grep h1
<h1>Welcome to nginx!</h1>
[demo@kworkhorse exercises]$
```

**Additional notes about LoadBalancer:**
* A service defined as LoadBalancer will still have some high-range port number assigned to it's main service port, just like NodePort. This has a clever purpose, but is an advance topic and is not our focus at this point.


# High Availability:

So far we have seen pods, deployments and services. We have also seen Kubernetes keeping up it's promise of resilience. Now we see how we can have **high availability** on Kubernetes. The easiest and preferred way to do this is by having multiple replicas for a deployment.


Lets increase the number of replicas of our nginx deployment to four(4):
```
$ kubectl scale deployment nginx --replicas=4
deployment "nginx" scaled
```

Check the deployment and pods:
```
$ kubectl get deployments
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
multitool   1         1         1            1           24m
nginx       4         4         4            4           34m
```

```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          24m
nginx-569477d6d8-4msf8       1/1       Running   0          20m
nginx-569477d6d8-bv77k       1/1       Running   0          34s
nginx-569477d6d8-s6lsn       1/1       Running   0          34s
nginx-569477d6d8-v8srx       1/1       Running   0          35s
```

**Notice:** The nginx deployment says Desired=4, Current=4, Available=4. And the pods also show the same. There are now 4 nginx pods running; one of them was already running (being older), and the other three are started just now.


You can also scale down! - e.g. to 2:

```
$ kubectl scale deployment nginx --replicas=2
deployment "nginx" scaled
```

```
$ kubectl get pods
NAME                         READY     STATUS        RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running       0          25m
nginx-569477d6d8-4msf8       1/1       Running       0          21m
nginx-569477d6d8-bv77k       0/1       Terminating   0          1m
nginx-569477d6d8-s6lsn       0/1       Terminating   0          1m
nginx-569477d6d8-v8srx       1/1       Running       0          2m
```

Notice that unnecessary pods are killed immediately.

```
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
multitool-3148954972-k8q06   1/1       Running   0          26m
nginx-569477d6d8-4msf8       1/1       Running   0          22m
nginx-569477d6d8-v8srx       1/1       Running   0          2m
```

You can delete the nginx deployment and service at this point. We have no use for these anymore. Besides, you can always re-create them.

```
$ kubectl delete deployment nginx

$ kubectl delete service nginx
```

## Small exercise for HA:
To prove that multiple pods of the same deployment provide high availability, we do a small exercise. To visualize it, we need to run a small web server which could return us some uniqe content when we access it. We will use our trusted multitool for it. Lets run it as a separate deployment and access it from our local computer.

```
$ kubectl run customnginx --image=praqma/network-multitool --replicas=4
deployment "customnginx" created
```

```
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
customnginx-3557040084-1z489   1/1       Running   0          49s
customnginx-3557040084-3hhlt   1/1       Running   0          49s
customnginx-3557040084-c6skw   1/1       Running   0          49s
customnginx-3557040084-fw1t3   1/1       Running   0          49s
```

Lets create a service for this deployment as a type=LoadBalancer: 

```
$ kubectl expose deployment customnginx --port=80 --type=LoadBalancer 
service "customnginx" exposed
```

Verify the service and note the public IP address:
```
$ kubectl get services
NAME          TYPE           CLUSTER-IP    EXTERNAL-IP        PORT(S)        AGE
customnginx   LoadBalancer   100.67.40.4   35.205.60.41       80:30087/TCP   1m
kubernetes    ClusterIP      100.64.0.1    <none>             443/TCP        17h
```

Query the service, so we know it works as expected:
```
$ curl -s 35.205.60.41 | grep IP
Container IP: 100.96.1.150 <BR></p>
```

Next, setup a small bash loop on your local computer to curl this IP address, and get it's IP address.
```
$ while true; do sleep 1; curl -s 35.205.60.41; done
Container IP: 100.96.2.36 <BR></p>
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.36 <BR></p>
^C
```

We see that when we query the LoadBalancer IP, it is giving us result/content from all four containers. None of the curl commands is timed out. Now, if we kill three out of four pods, the service should still respond, without timing out. We let the loop run in a separate terminal, and kill three pods of this deployment from another terminal. 

```
$ kubectl delete pod customnginx-3557040084-1z489 customnginx-3557040084-3hhlt customnginx-3557040084-c6skw 
pod "customnginx-3557040084-1z489" deleted
pod "customnginx-3557040084-3hhlt" deleted
pod "customnginx-3557040084-c6skw" deleted
```

Immediately check the other terminal for any failed curl commands or timeouts. 
```
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.1.149 <BR></p>
Container IP: 100.96.1.149 <BR></p>
Container IP: 100.96.1.150 <BR></p>
Container IP: 100.96.2.36 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.37 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.2.38 <BR></p>
Container IP: 100.96.1.151 <BR></p>
```

We notice that no curl command failed, and actually we have started seeing new IPs. Why is that? It is because, as soon as the pods are deleted, the deployment sees that it's desired state is four pods, and there is only one running, so it immediately starts three more to reach that desired state. And, while the pods are in process of starting, one surviving pod takes the traffic.

```
[kamran@kworkhorse kubernetes-katas]$ kubectl get pods
NAME                           READY     STATUS        RESTARTS   AGE
customnginx-3557040084-0s7l8   1/1       Running       0          15s
customnginx-3557040084-1z489   1/1       Terminating   0          16m
customnginx-3557040084-3hhlt   1/1       Terminating   0          16m
customnginx-3557040084-bvtnh   1/1       Running       0          15s
customnginx-3557040084-c6skw   1/1       Terminating   0          16m
customnginx-3557040084-fw1t3   1/1       Running       0          16m
customnginx-3557040084-xqk1n   1/1       Running       0          15s
[kamran@kworkhorse kubernetes-katas]$ 
```
This proves, Kubernets provides us High Availability, using multiple replicas of a pod. 



------ 



# Namespaces:
Namespaces are the default way for Kubernetes to separate resources, and most Kubernetes resources such as pods and deployments must belong to a namespace. Namespaces is a quite powerful concept, but not unusual, as in computing - we are used to having isolated environments such as home directories, jailed environments, etc. Kubernetes clusters come with a namespace called **default**. However, namespaces in Kubernetes are not necessarily strongly separated and e.g. by default Kubernetes allows network connectivily between pods in different namespaces.

When you execute a kubectl command without specifying a namespace, it is run in/against the namespace named **default** !. So far all the commands you have executed above, have been executed in the *default* namespace. You can optionally use the namespace flag (-n mynamespace) to execute the command a specific namespace. When you are creating Kubernetes objects though *yaml* files, you can specify a namespace for a particular resource. 

Note: In case, you are on a shared cluster for this workshop, then watch out for which namespace you are using to deploy your objects in. 

It is also possible to limit namespaces and resources at user level, but it would be a bit too involved for your first experience with Kubernetes

## Create a namespace

Use the following syntax to create a namespace and to list objects from a namespace:
```
kubectl create namespace <name>

kubectl get pods -n <name>
```

Here are few examples:
```
$ kubectl create namespace dev
namespace "dev" created

$ kubectl create namespace test
namespace "test" created

$ kubectl create namespace prod
namespace "prod" created
```

```
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    2d
dev           Active    12s
kube-system   Active	2d
prod          Active    5s
test          Active    8s
```


The below commands do the same thing, because kubernetes commands will default to the *default* namespace:

```
kubectl get pods -n default
kubectl get pods
```

## Manage Kubernetes objects in a namespace:

So let's try to create an nginx deployment in the *dev* namespace.

```
$ kubectl --namespace=dev run nginx --image=nginx
deployment "nginx" created

$ kubectl get pods -n dev
NAME                     READY     STATUS    RESTARTS   AGE
nginx-4217019353-fk9ph   1/1       Running   0          10s
```

So, you get the idea. You include `-n <namespacename>` in any kubectl invocation, and it will operate on the given namespace only. You can also use `--all-namespaces=true" to get objects from all namespaces.

The yaml files for the coming exercises will give you a better impression of what is sensible.

This concludes the exercise, happy coding!

------------------

## Useful commands

    kubectl config current-context
    kubectl config use-context docker-for-desktop
    kubectl version
    kubectl cluster-info
    kubectl get nodes
    kubectl get all
    kubectl describe
