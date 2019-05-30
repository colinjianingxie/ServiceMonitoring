# ServiceMonitoring

This guide will teach you how to monitor services in a Kubernetes cluster with Prometheus and Grafana.
In addition, this guide will also show you how to create a Go app that exposes the /metrics endpoint.
For those who want to save time, pull the documents from this repository.

## Pre-requisites
- Minikube installed
- Have a Dockerhub account
- Install Go Lang (Optional)

## Step 1: Create a working directory

You will be working in a specified directory throughout this tutorial. 
For this tutorial, we will be working in the **Downloads/workspace** folder. 

1. Go to the local Downloads Folder
2. Create a folder called **workspace**.

## Step 2: Creating a Custom Go App and Deploying onto Dockerhub (Optional)

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
8. The server will now run. To run the demo site, navigate to: **localhost:8080** in your browser. To get the metrics, go to ...**localhost:8080/metrics** in your browser.  

Now we need to create a docker image of the hello Go App we just created and upload the image onto Dockerhub.
1. Create a Dockerfile inside the **Downloads/workspace/src/hello** folder. The file shouldn't have an extension and should be ...called Dockerfile, case sensitive. 
2. Inside the Dockerfile write the following:
...**Note: You might need to edit the file Maintainer with your own name and email**
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
...Example: **docker build . -t xienokia/hello-world**
5. Login to your Dockerhub by running the command: **docker login**
6. Now, push the image by running: **docker push (insert Dockerhub username)/(insert Dockerhub repository)**
...Example: **docker push xienokia/hello-world**
7. We will be using the image off of ***xienokia/hello-world*** for this tutorial

## Step 3: Setting up the Kubernetes Cluster

Now we need to setup the Kubernetes Cluster inside Minikube. The process will include the following:
1. Starting minikube
2. Creating custom namespaces
3. Deploying services
4. Deploying Prometheus





