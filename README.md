# Service Monitoring (Mac OS)

This guide will teach you how to monitor services in a Kubernetes cluster with Prometheus and Grafana.
In addition, this guide will also show you how to create a Go app that exposes the /metrics endpoint.
***For those who want to save time, pull the documents from this repository. This tutorial will include the creation of the files in this repository***

# Table of Contents
0. [Pre-requisites](#Prereq)
1. [Create a working directory](#working)
2. [Creating Go App (Optional)](#go)
3. [Setting up the Kubernetes Cluster and Prometheus](#kubernetes)
4. [Connecting Grafana](#grafana)

## Pre-requisites <a name="prereq"></a>
- Basic Kubernetes knowledge
- Minikube installed
- Have a Dockerhub account
- Install Go Lang (Optional)

## Step 1: Create a working directory <a name="working"></a>

You will be working in a specified directory throughout this tutorial. 
For this tutorial, we will be working in the **Downloads/workspace** folder. 

1. Go to the local Downloads Folder
2. Create a folder called **workspace**.

## Step 2: Creating a Custom Go App and Deploying onto Dockerhub (Optional) <a name="go"></a>

If you do not want to create a custom Go App to expose metrics, feel free to use the Docker image (xienokia/hello-app) located on [Dockerhub](https://hub.docker.com/r/xienokia/hello-app) and proceed to Step 3.

1. Inside the **workspace** folder, create a **src** folder
2. Inside the **src** folder, create a **hello** folder
3. Inside the **hello** folder, create a **hello.go** file
4. Edit and save the **hello.go** file accordingly:

```
package main

import (
	"flag"
	"log"
	"net/http"
	"os"
	"fmt"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	version = prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "version",
		Help: "Version information about this binary",
		ConstLabels: map[string]string{
			"version": "v0.1.0",
		},
	})
	httpRequestsTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
		Name: "http_requests_total",
		Help: "Count of all HTTP requests",
	}, []string{"code", "method"})
)

func main() {
	dir, err := os.Getwd()
	if err != nil {
		log.Fatal(err)
	}
  	fmt.Println(dir)

	bind := ""
	flagset := flag.NewFlagSet(os.Args[0], flag.ExitOnError)
	flagset.StringVar(&bind, "bind", ":8080", "The socket to bind to.")
	flagset.Parse(os.Args[1:])

	r := prometheus.NewRegistry()
	r.MustRegister(httpRequestsTotal)
	r.MustRegister(version)

	handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("Hello from example application."))
	})
	notfound := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusNotFound)
	})
	http.Handle("/", promhttp.InstrumentHandlerCounter(httpRequestsTotal, handler))
	http.Handle("/err", promhttp.InstrumentHandlerCounter(httpRequestsTotal, notfound))

	http.Handle("/metrics", promhttp.HandlerFor(r, promhttp.HandlerOpts{}))
	log.Fatal(http.ListenAndServe(bind, nil))
}
```

5. Navigate inside the Terminal to the **Downloads/workspace/src/hello** folder
6. Run the following commands: **go build**
7. By running that, it will create a build file, which to compile, run the following command: **./hello**
8. The server will now run. To run the demo site, navigate to: **localhost:8080** in your browser. To get the metrics, go to **localhost:8080/metrics** in your browser.  

Now we need to create a docker image of the hello Go App we just created and upload the image onto Dockerhub.
1. Create a Dockerfile inside the **Downloads/workspace/src/hello** folder. The file shouldn't have an extension and should be called Dockerfile, case sensitive. 
2. Inside the Dockerfile write the following:
**Note: You might need to edit the file Maintainer with your own name and email**
```

# Dockerfile References: https://docs.docker.com/engine/reference/builder/
# Start from golang v1.11 base image
FROM golang:1.12

# Add Maintainer Info
LABEL maintainer="inputnamehere <input_email@here>"

# Set the Current Working Directory inside the container
WORKDIR $GOPATH/src/hello/hello

# Copy everything from the current directory to the PWD(Present Working Directory) inside the container
COPY . .

# Download all the dependencies
# https://stackoverflow.com/questions/28031603/what-do-three-dots-mean-in-go-command-line-invocations
RUN go get -d -v ./...

# Install the package
RUN go install -v ./...

# This container exposes port 8080 to the outside world
EXPOSE 8080

# Run the executable
CMD ["hello"]

```
3. Inside the terminal, go to the **Downloads/workspace/src/hello** folder
4. Create the image by running the command: **docker build . -t (insert Dockerhub username)/(insert Dockerhub repository)**
Example: **docker build . -t xienokia/hello-world**
5. Login to your Dockerhub by running the command: **docker login**
6. Now, push the image by running: **docker push (insert Dockerhub username)/(insert Dockerhub repository)**
Example: **docker push xienokia/hello-world**
7. We will be using the image off of **xienokia/hello-world** for this tutorial

## Step 3: Setting up the Kubernetes Cluster and Prometheus <a name="kubernetes">

Now we need to setup the Kubernetes Cluster inside Minikube. The process will include the following:
1. Starting minikube
2. Creating custom namespaces
3. Deploying services
4. Deploying Prometheus Operator and Service Accounts
5. Deploying and exposing Prometheus
6. Deploying service monitors

### 1. Starting Minikube

1. Navigate to the **Downloads/workspace** folder through the Terminal
2. Type the command: **minikube start**
3. Note: to stop Minikube, type: **minikube stop** and to delete Minikube, type: **minikube delete**

### 2. Creating custom namespaces

1. Create a namespace called **test** by typing the command: **kubectl create namespace test**
2. To verify you have created the **test** namespace, type the command: **kubectl get namespaces**
**test** should appear as one of the namespaces.

### 3. Deploying services

1. Make sure you are still in the **Downloads/workspace** folder (or your designated workspace folder)
2. Copy the following:
```
kind: Service
apiVersion: v1
metadata:
  name: example-service-test
  namespace: test
  labels:
    app: example-service-test
spec:
  selector:
    app: example-service-test
  ports:
  - name: web
    port: 8080
    nodePort: 30901
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: example-service-test
  namespace: test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: example-service-test
    spec:
      containers:
      - name: example-service-test
        image: xienokia/hello-app
        ports:
        - name: web
          containerPort: 8080
```
and save into a **sample.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).
2. Apply the **sample.yaml** file in minikube by typing: **kubectl apply -f sample.yaml**
Note: if you chose to save the yaml file into a custom location, type the command: **kubectl apply -f (sample.yaml file location)**
3. Make sure the pod is working by running: **kubectl -n test get pods**
4. To check if the service is working inside the namespace type: **kubectl -n test get svc** 
5. To see the result of the deployment, type: **kubectl cluster-info**. Observe the "Kubernetes Master is running at..." IP address. In a browser, type the following: **(Kubernetes Master IP address):30901**. For example: (192.168.99.129:30901). 
6. To go to the metrics, type the following in the address bar: **(Kubernetes Master IP address):30901/metrics**. For example: (192.168.99.129:30901/metrics). You should see the *http_requests* metric increase as traffic increases to the site.

### 4. Deploying Prometheus Operator and Service Accounts <a name="prometheus-operator"></a>

The Prometheus Operator makes the Prometheus configuration Kubernetes native and manages and operates Prometheus and Alertmanager clusters. For more information, please check out the [prometheus operator guide](https://github.com/coreos/prometheus-operator).

1. Copy the following:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    apps.kubernetes.io/component: controller
    apps.kubernetes.io/name: prometheus-operator
    apps.kubernetes.io/version: v0.29.0
  name: prometheus-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-operator
subjects:
- kind: ServiceAccount
  name: prometheus-operator
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    apps.kubernetes.io/component: controller
    apps.kubernetes.io/name: prometheus-operator
    apps.kubernetes.io/version: v0.29.0
  name: prometheus-operator
rules:
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - '*'
- apiGroups:
  - monitoring.coreos.com
  resources:
  - alertmanagers
  - prometheuses
  - prometheuses/finalizers
  - alertmanagers/finalizers
  - servicemonitors
  - prometheusrules
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
  - delete
- apiGroups:
  - ""
  resources:
  - services
  - services/finalizers
  - endpoints
  verbs:
  - get
  - create
  - update
  - delete
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    apps.kubernetes.io/component: controller
    apps.kubernetes.io/name: prometheus-operator
    apps.kubernetes.io/version: v0.29.0
  name: prometheus-operator
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      apps.kubernetes.io/component: controller
      apps.kubernetes.io/name: prometheus-operator
  template:
    metadata:
      labels:
        apps.kubernetes.io/component: controller
        apps.kubernetes.io/name: prometheus-operator
        apps.kubernetes.io/version: v0.29.0
    spec:
      containers:
      - args:
        - --kubelet-service=kube-system/kubelet
        - --logtostderr=true
        - --config-reloader-image=quay.io/coreos/configmap-reload:v0.0.1
        - --prometheus-config-reloader=quay.io/coreos/prometheus-config-reloader:v0.29.0
        image: quay.io/coreos/prometheus-operator:v0.29.0
        name: prometheus-operator
        ports:
        - containerPort: 8080
          name: http
        resources:
          limits:
            cpu: 200m
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
      nodeSelector:
        beta.kubernetes.io/os: linux
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
      serviceAccountName: prometheus-operator
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    apps.kubernetes.io/component: controller
    apps.kubernetes.io/name: prometheus-operator
    apps.kubernetes.io/version: v0.29.0
  name: prometheus-operator
  namespace: default
```
and save into a **prometheus-operator.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).

2. Apply the **prometheus-operator.yaml** file in minikube by typing: **kubectl apply -f prometheus-operator.yaml**
3. Now, Copy the following:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default
```
and save into a **cluster.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).

4. Apply the **cluster.yaml** file in minikube by typing: **kubectl apply -f cluster.yaml**

### 5. Deploying and exposing Prometheus 

Prometheus is a program that can scrape metrics off of various Kubernetes objects. 

1. Copy the following:
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  serviceMonitorNamespaceSelector: {} # auto discovers all namespaces
  serviceMonitorSelector: {} # auto discovers all monitors configured one line above
  resources:
    requests:
      memory: 400Mi
  enableAdminAPI: false
```
and save into a **prometheus.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).

2. Apply the **prometheus.yaml** file in minikube by typing: **kubectl apply -f prometheus.yaml**
3. Now, Copy the following:

```
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  type: NodePort
  ports:
  - name: web
    nodePort: 30900
    port: 9090
    protocol: TCP
    targetPort: web
  selector:
    prometheus: prometheus
```
and save into a **expose.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).

4. Apply the **expose.yaml** file in minikube by typing: **kubectl apply -f expose.yaml**
Doing so will help expose Prometheus to the 30900 port (specified in the **expose.yaml** file)
5. To check if Prometheus is installed correctly, type: **kubectl cluster-info** to obtain the Kubernetes Master IP. 
6. In the browser, type: **(Kubernetes Master IP):30900** and Prometheus should pop up. For example: **192.168.99.129:30900**

### 6. Deploying Service Monitors 

To monitor the services in each namespace, we need to add a service monitor to the namespace. 

1. Copy the following:
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-service-test
  labels:
    team: frontendtest1
  namespace: test
spec:
  selector:
    matchLabels:
      app: example-service-test
  endpoints:
  - port: web

```
and save into a **service-monitors.yaml** file into the **Downloads/workspace** folder (or wherever you choose to save it).

2. Apply the **service-monitors.yaml** file in minikube by typing: **kubectl apply -f service-monitors.yaml**
3. We should now be able to see the service monitors in Prometheus. Go to Prometheus by typing: **(Kubernetes Master IP):30900** in the browser. 
4. Click **Status -> Service Discovery** in the menu bar
5. Make sure the service is being discovered by the service monitor
6. Click **Status -> Targets** in the menu bar
7. Make sure the Targeted service monitor is being shown.
8. If all goes well, click **Graph** in the menu bar, and type the query: **http_requests_total**
9. Going to the app should now trigger http requests, which prometheus can show

## Setting up Grafana <a name="grafana"></a>

Grafana takes Prometheus and displays a more appealing chart.

### Install Grafana
1. Install Grafana by running: **docker run -d --name=grafana -p 3000:3000 grafana/grafana**
2. Now, access Grafana by going to your browser URL and typing: **http://localhost:3000**
The default login is:
username: admin
password: admin

### Creating Prometheus Data Source
1. Click on the Grafana Logo to open the sidebar menu
2. Click on **Data Sources** in the side bar
3. Click **Add New**
4. Select **Prometheus** as the type of data source
5. Type the appropriate Prometheus server URL: **(Kubernetes Master IP):30900**
6. Finish by clicking **Add** to save the data source

### Creating the Grafana Dashboard
1. Click the **+** sign on the left panel
2. Click **Dashboard**
3. Click **Add query**
4. For the query, type the query that was exposed in the metrics (**http_requests_total**).
5. To edit the title, click the settings and change the title accordingly

## Congratulations! You should now have Grafana monitoring services from your namespaces!





