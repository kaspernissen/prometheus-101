# Prometheus demo

This is a simple demo on how to get started playing with Prometheus and Grafana locally in Kubernetes


## Getting a cluster
There's plenty of different options for setting up a cluster. 

This demo has been tested with both `minikube` and `kind`.

### minikube
To install `minikube` follow the instructions here: https://kubernetes.io/docs/tasks/tools/install-minikube/

To run `minikube`, just run the following command:

```
$ minikube start
```

### kind
To install `kind` follow the instructions here: https://kind.sigs.k8s.io/docs/user/quick-start/ 

To run `kind` execute the following command:

```
$ kind create cluster
```
If you want a more advance cluster setup, I have added an example of how to spin up a cluster with a master node and three worker nodes

```
$ kind create cluster --name threeworkers --config kubernetes/kind/cluster.yaml
```


## Installing helm
In order to get things set up quickly we will be using `helm`.

Follow the instructions here to install `helm`: https://github.com/helm/helm

Initialize `helm` by calling `init`. This installs a service in the `minikube` cluster called `tiller` used by the `helm` client to interact with the kubernetes api.
```
$ helm init
```

There's a little caveat when using `kind`. You have to set up `helm` with rbac permissions. You can do that easily by appling the following `ServiceAccount` and `ClusterRoleBinding` to your cluster.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```
As you can see above, this will provide `helm` with cluster-admin privileges. To apply this to the cluster, run:
```
$ kubectl apply -f kubernetes/helm/rbac.yaml
```
Now run the following command to setup `helm` with the correct ´ServiceAccount`. 

```
helm init --service-account tiller
```

## Installing Prometheus

Next, we install `prometheus`. It is as simple as:

```
$ helm install stable/prometheus --name prometheus
```

The output of this command:

```
$ kubectl get pods
NAME                                             READY     STATUS    RESTARTS   AGE
prometheus-alertmanager-6db559d695-kfw95         2/2       Running   0          3m54s
prometheus-kube-state-metrics-5db7466b57-jm9v7   1/1       Running   0          3m54s
prometheus-node-exporter-xljv2                   1/1       Running   0          3m54s
prometheus-pushgateway-59bd88779d-clrnh          1/1       Running   0          3m54s
prometheus-server-5f74c4749-xbrrk                2/2       Running   0          3m54s
```

`helm` will also output a list of notes on how you access the different services: 
```
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.default.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.default.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9093


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.default.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace default port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

As you can see, we get the following services

* Prometheus Server
* Prometheus Alertmanager
* Prometheus Pushgateway
* Node-exporter
* Kube-state-metrics

Our Prometheus setup is now running and collecting metrics from the minikube cluster.


## Installing Grafana

Now, that we have `prometheus` running, we want a tool to create awesome graphs, and display our data in a useful way. `grafana` is a open-source metrics visualization tool.

Installing grafana, is as simple as:

```
$ helm install stable/grafana --name grafana
```

Again, `helm` will output useful commands for how you access grafana.
```
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   grafana.default.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:

     export POD_NAME=$(kubectl get pods --namespace default -l "app=grafana,release=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace default port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

## Connecting the dots, e.g. grafana and prometheus

Open grafana as described in the NOTES, using a port-forward. Go to your browser and enter http://localhost:3000.

Login using the password obtained from running the command in the notes, for convinience, the same command can be seen below. (username is admin)

```
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Go to Configuration > Data Sources
![](docs/1-configuration_datasource.png)

Press the big green button saying "Add data source"
![](docs/2-add_datasource.png)

Select Promethes as the new data source to be added.
![](docs/3-select-prometheus.png)

Connect to the Promethes server, by entering the name of the kubernetes service for the Prometheus server and the port. You can verify this information as:

```
$ kubectl get svc
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
grafana                         ClusterIP   10.105.170.50    <none>        80/TCP     24m
kubernetes                      ClusterIP   10.96.0.1        <none>        443/TCP    6d22h
prometheus-alertmanager         ClusterIP   10.97.24.144     <none>        80/TCP     24m
prometheus-kube-state-metrics   ClusterIP   None             <none>        80/TCP     24m
prometheus-node-exporter        ClusterIP   None             <none>        9100/TCP   24m
prometheus-pushgateway          ClusterIP   10.102.210.218   <none>        9091/TCP   24m
prometheus-server               ClusterIP   10.99.2.33       <none>        80/TCP     24m
```
As you can see the `prometheus-server` is the name and the port is 80. We therefore enter the following in grafana:

```
http://prometheus-server
```

![](docs/4-connect-to-server.png)

Explore your metrics using the `Explore` in grafana
![](docs/5-explorer.png)


## Removing it all again

```
$ helm del --purge prometheus
$ helm del --purge grafana
```

## Alternative Prometheus installation: Prometheus Operator

```
$ helm install stable/prometheus-operator --name prometheus
```

This will install Prometheus using a Custom Resource Object (CRD) in Kubernetes. The `Prometheus` object installed by default looks like this:

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  creationTimestamp: 2019-05-08T05:58:04Z
  generation: 1
  labels:
    app: prometheus-operator-prometheus
    chart: prometheus-operator-5.5.1
    heritage: Tiller
    release: prometheus
  name: prometheus-prometheus-oper-prometheus
  namespace: default
  resourceVersion: "78362"
  selfLink: /apis/monitoring.coreos.com/v1/namespaces/default/prometheuses/prometheus-prometheus-oper-prometheus
  uid: 3ed7a04e-7156-11e9-ae1f-080027554f4a
spec:
  alerting:
    alertmanagers:
    - name: prometheus-prometheus-oper-alertmanager
      namespace: default
      pathPrefix: /
      port: web
  baseImage: quay.io/prometheus/prometheus
  externalUrl: http://prometheus-prometheus-oper-prometheus.default:9090
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
    matchLabels:
      app: prometheus-operator
      release: prometheus
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-prometheus-oper-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
    matchLabels:
      release: prometheus
  version: v2.9.1
```

Based on this CRD the operator will configure and setup Prometheus.

To connect this installation with Grafana, as we did before we have to figure out how to access the prometheus server.

```
$ kubectl get svc
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,6783/TCP   7m8s
prometheus-grafana                        ClusterIP   10.97.44.173     <none>        80/TCP              5m49s
prometheus-kube-state-metrics             ClusterIP   10.105.156.226   <none>        8080/TCP            5m49s
prometheus-operated                       ClusterIP   None             <none>        9090/TCP            5m34s
prometheus-prometheus-node-exporter       ClusterIP   10.105.64.5      <none>        9100/TCP            5m49s
prometheus-prometheus-oper-alertmanager   ClusterIP   10.103.13.92     <none>        9093/TCP            5m49s
prometheus-prometheus-oper-operator       ClusterIP   10.105.249.207   <none>        8080/TCP            5m49s
prometheus-prometheus-oper-prometheus     ClusterIP   10.108.147.172   <none>        9090/TCP            5m49s
```
The service we should connect to is called: prometheus-prometheus-oper-prometheus and exposes port 9090, e.g. you can access it using `http://prometheus-prometheus-oper-prometheus:9090` from grafana.

As you can see in the output above. The Prometheus Operator actually comes with Grafana bundled as well. To access this, you can do it in the same way as before, using a port-forward to the pod. In my case, it is as follows. (change to the pod-name running in your cluster)

```
$ kubectl port-forward prometheus-grafana-8589b465b9-2t7bc 3000 
```

The default username and password are:

```
Username: admin
Password: prom-operator
```

Grafana is already setup with the correct data source to the prometheus server, along with a bunch of preloaded dashboards to explore.


## Putting load on the system

We can play around with putting some load on the system using the `resource-consumer` image. [link](https://github.com/kubernetes/kubernetes/tree/master/test/images/resource-consumer)

```
kubectl run resource-consumer --image=gcr.io/kubernetes-e2e-test-images/resource-consumer:1.4 --port 8080 --requests='cpu=500m,memory=256Mi'
```

Now you can easily port-forward the pod to `localhost``

```
kubectl port-forward <resource consumer pod> 8080
```

Now we can simply `curl` the resource-consumer to start consuming resources:

```
curl --data "megabytes=30&durationSec=600" http://localhost:8080/ConsumeMem
```

If you have used the Prometheus Operator setup, you will be able to go the Pod dashboard and watch the memory consumption of the `resource-consumer` pod rise.




# TODO:
* Add examples of interacting with the Alertmanager and Pushgateway
