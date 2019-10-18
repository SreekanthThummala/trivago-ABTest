# A/B Test Infrastructure
A/B testing deployments consists of routing a subset of users to a new
functionality under specific conditions. It is usually a technique for making
business decisions based on statistics rather than a deployment strategy.
However, it is related and can be implemented by adding extra functionality to a
canary deployment.

## Basic Steps
These are roughly the steps you need to follow to get an Istio enabled ‘trivago’ app:
1. Create a Kubernetes cluster and install Istio with automatic sidecar injection.
1. Create a Docker image out of the files provided and push it to a public image repository.
1. Create Kubernetes Deployment and Service for your container.
1. Create a Gateway to enable HTTP traffic to your cluster.
1. Create a VirtualService to expose Kubernetes Service via Gateway.
1. Create multiple versions of your app, create a DestinationRule to define subsets with different weights that you can refer from the VirtualService.

## Pre-requsites 

   Before starting, it is recommended to know the basic concept of the
   [Istio routing API](https://istio.io/blog/2018/v1alpha3-routing/).

   ### Deploy Istio

   In this example, Istio 1.0.0 is used. To install Istio, follow the
   [instructions](https://istio.io/docs/setup/kubernetes/helm-install/) from the
   Istio website.

   Automatic sidecar injection should be enabled by default. Then annotate the
   default namespace to enable it.

   ```
   $ kubectl label namespace default istio-injection=enabled
   ```
   
## Package the trivago app in a docker container

   Next, prepare your app to run as a container. The first step is to define the container and its contents.

   In the base directory of the app, I have already created a Dockerfile to define the Docker image.
   Make sure you replace YOUR-PROJECT-ID with the id of your project

## Build golang image

  Please run the below commands to build the image and push to docker hub.

   ```
   $cd trivago-golang
   ```

   Now, let's build the image:

   ```
   docker build -t docker.io/YOUR-PROJECT-ID/trivago-golang:v1
   ```

   Once this completes (it'll take some time to download and extract everything), you can see the image is built and saved locally:

   ```
   docker images
   REPOSITORY                                  TAG   
   docker.io/yourproject-XXXX/trivago-golang   v1 
   ```
   Test the image locally with the following command which will run a Docker container locally on port 8080 from your newly-created container image:

   ```
   docker run -p 8080:8080 docker.io/YOUR-PROJECT-ID/trivago-golang:v1
   ```

   And push the Image to Container Registry:

```docker push docker.io/YOUR-PROJECT-ID/trivago-golang:v1
```
## Build java Image

   Please run the below commands to build the image and push to docker hub.

   ```
   $cd trivago-java
   ```

   Now, let's build the image:

   ```
   docker build -t docker.io/YOUR-PROJECT-ID/trivago-java:v2
   ```

   Once this completes (it'll take some time to download and extract everything), you can see the image is built and saved locally:

   ```
   docker images
   REPOSITORY                                  TAG   
   docker.io/yourproject-XXXX/trivago-golang   v1
   docker.io/yourproject-XXXX/trivago-java     v2 
   ```
   Test the image locally with the following command which will run a Docker container locally on port 8080 from your newly-created container image:

   ```
   docker run -p 8080:8080 docker.io/YOUR-PROJECT-ID/trivago-java:v2
   ```
  And push The Image to the Container Registry:

```docker push docker.io/YOUR-PROJECT-ID/trivago-java:v2
```

## Deployment and Service
As mentioned, the app lifecycle is managed by Kubernetes. Therefore, you need to start with creating a Kubernetes Deployment and Service. 
In this case, I have a containerized the apps whose image I already pushed to Docker Hub. 

I have created the deployment and service files under AB-Testing with the name trivago.yaml.

Make sure you replace YOUR-PROJECT-ID with the id of your project

Run the below command to create the deployment and service.

```
$ kubectl apply -f trivago.yaml

service "trivago-service" created
deployment.extensions "trivago-v1" created
deployment.extensions "trivago-v2" created
```

## Gateway
We can now start looking into Istio Routing. First, we need to enable HTTP/HTTPS traffic to our service mesh. To do that, we need to create a Gateway. Gateway describes a load balancer operating at the edge of the mesh receiving incoming or outgoing HTTP/TCP connections.
I had created an trivago-gateway.yaml file.

Create the Gateway:

```
$ kubectl apply -f trivago-gateway.yaml

gateway.networking.istio.io "trivago-gateway" created
```

At this point, we have HTTP traffic enabled for our cluster. We need to map the Kubernetes Service we created earlier to the Gateway. We’ll do that with a VirtualService.

## VirtualService
A VirtualService essentially connects a Kubernetes Service to Istio Gateway. It can define set of traffic routing rules to apply when a host is addressed.
Let’s create an trivago-virtualservice.yaml file:

Notice that a VirtualService is tied to a specific Gateway and it defines a host that refers to the Kubernetes Service.

Create the VirtualService:

```
$ kubectl apply -f trivago-virtualservice.yaml

virtualservice.networking.istio.io "trivago-virtualservice" created
```

## Test the application
We’re ready to test our app. We need to get the IP address of the Istio Ingress Gateway:

```$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP                                                                                                        
istio-ingressgateway   LoadBalancer   10.31.247.41   35.240.XX.XXX
```

When we browse to the EXTERNAL-IP, we should see the Trivago app.
This is because both golang(v1) and java(v2) deployments are exposed behind the same Kubernetes service (trivago-service) and the VirtualService you created in the previous lab (trivago-virtualservice) uses that service as a host.


## Split traffic between versions

We want to split traffic between versions for golang and java. We want to send 70% of the traffic to the v1 and 30% of the traffic to the v2 version of the service. You can easily achieve this with Istio. Created a new trivago-virtualservice-weights.yaml file to refer to the two subsets with different weights:

Update the VirtualService:

```
kubectl apply -f trivago-virtualservice-weights.yaml
```

Now, when you refresh the browser, you should see the v1 vs. v2 versions served with roughly a 7:3 ratio.

## Cleanup
You can delete the app and uninstall Istio or you can simply delete the Kubernetes cluster.

### Delete the application
To delete the application:

```kubectl delete -f trivago-gateway.yaml
kubectl delete -f trivago-virtualservice.yaml
kubectl delete -f trivago-destinationrule.yaml
kubectl delete -f trivago.yaml
kubectl delete -f trivago-virtualservice-weights.yaml
```

To confirm that the application is gone:

```kubectl get gateway 
kubectl get virtualservices
kubectl get destinationrule
kubectl get pods
```
