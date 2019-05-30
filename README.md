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
8. The server will now run. To run the demo site, navigate to: **localhost:8080** in your browser. To get the metrics, go to **localhost:8080/metrics** in 	your browser.  
9. 


