# Introduction
This blog covers deploying the Contour ingress controller and demonstrates its use with a Ingress and IngressRoute examples.  The Contour ingress controller will be deployed on my local system which does not have access to a service of type LoadBalancer, so the Contour service is modified to be of type NodePort.

## Deploy Contour
There are a few different methods for deploying Contour, but I like to just clone the github repository and deploy from the YAMLs provided.
```console
$ git clone https://github.com/heptio/contour.git
```
```console
$ cd contour/deployment/deployment-grpc-v2
```
```console
$ ls -1
01-common.yaml
02-contour.yaml
02-rbac.yaml
02-service.yaml
```
As you can see there is a 02-service.yaml file.  We will be modifying this file to make the service of type NodePort rather than LoadBalancer.  If you are on a Kubernetes implementation like PKS with NSX-T or one of the public cloud providers like GKE you would not need to do this.
```console
apiVersion: v1
kind: Service
metadata:
 name: contour
 namespace: heptio-contour
 annotations:
  # This annotation puts the AWS ELB into "TCP" mode so that it does not
  # do HTTP negotiation for HTTPS connections at the ELB edge.
  # The downside of this is the remote IP address of all connections will
  # appear to be the internal address of the ELB. See docs/proxy-proto.md
  # for information about enabling the PROXY protocol on the ELB to recover
  # the original remote IP address.
  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp

  # Scrape metrics for the contour container
  # The envoy container is scraped by annotations on the pod spec
  prometheus.io/port: "8000"
  prometheus.io/scrape: "true"
spec:
 ports:
 - port: 80
   nodePort: 30080 # or whatever NodePort value you want within your K8S allowed range
   name: http
   protocol: TCP
   targetPort: 8080
 - port: 443
   nodePort: 30443 # or whatever NodePort value you want within your K8S allowed range
   name: https
   protocol: TCP
   targetPort: 8443
 selector:
   app: contour
 type: NodePort
---
```
Now save and apply the resources to your Kubernetes environment.
```console
$ kubectl apply -f .
```

## Verify Contour is Running
The deployment above will create a new heptio-contour namespace with various Kubernetes resources.  One of which should be a service of type NodePort pointing to Contour pods running instances of the Contour container and the Envoy container.  Make sure both containers are in the ready state.  Given we have a service of type NodePort, you should be able to hit any of the nodes in your cluster to access it.  Note: Contour does not provide any ingress traffic until an Ingress or IngressRoute is actually deployed.
```console
$ kubectl get pods -n heptio-contour
NAME                       READY   STATUS    RESTARTS   AGE
contour-66bc464fb5-f4b7b   2/2     Running   2          7d4h
contour-66bc464fb5-nkwgd   2/2     Running   2          7d4h
```

```console
$ kubectl get svc -n heptio-contour
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
contour                    NodePort   10.107.138.251   <none>        80:30080/TCP,443:30443/TCP   7d4h
```

## Deploy Examples
Contour supports the basic Ingress resource type as well as a CustomResourceDefinition type IngressRoute.  Below are some basic examples of each.  I encourage you to look into the IngressRoute type.  It provides some cool features like weighted service traffic.

### Deploy a Service
First thing we will need is a service that our ingress traffic can connect.  I am using the term service here in the generic sense like a web-service.  Simplest way is just to create a deployment of nginx and then expose it using a Kubernetes Service resource.
```console
$ kubectl run --image=nginx web-dep
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --gene
rator=run-pod/v1 or kubectl create instead.
deployment.apps/web-dep created
```
Don't worry about the warning message.  The deployment should be deployed.  
Next, expose the Deployment as a Service.
```console
$ kubectl expose deployment web-dep --port=80
service/web-dep exposed
```
That will create a Kubernetes Service of type ClusterIP which is fine for our examples.  

### Deploy Ingress Resource
Now we want to create a Kubernetes resource of type Ingress.  The example below is a simple example that routes traffic to our nginx web service.  
```console
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ing1
  annotations:
    kubernetes.io/ingress.class: contour

spec:
  rules:
  - host: s1.cluster1.corp.local
  - http:
      paths:
      - path: /
        backend:
          serviceName: web-dep
          servicePort: 80
```
Save the YAML to a file and apply it to your cluster.  Once it is deployed execute the following curl command.
```console
$ curl -v -H 'Host: s1.cluster1.corp.local' worker1:30080
```
Few items to point out with the Ingress resource and curl command.
* host is nothing more than something I made up since it is on my local system and I update the header when I make the actual curl call.  This makes it easy for testing.
* Worker1 is one of the nodes in my Kubernetes cluster and has an entry in /etc/hosts.
* Path is to the root of the host; "/"
* Backend is the service that we exposed previously.

