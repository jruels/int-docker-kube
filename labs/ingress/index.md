# Ingress on EKS
## Overview 
To use an ingress on Kubernetes you must have an Ingress controller and ingress routing rules. As there are different ingress controllers that can do this job, it’s important to choose the right one for the type of traffic and load coming into your Kubernetes cluster. In this lab you learn how to use an NGINX ingress controller on Amazon EKS, and how to front-face it with a Network Load Balancer (NLB).

![](index/k6ZY9iBELZuCJCXTR-BBAg%202.png)

Kubernetes ingress provides the following features: 
* Content-based routing: e.g.,  routing based on HTTP method, request headers, or other properties of the specific request.
* Resilience: e.g., rate  limiting, timeouts.
* Support for multiple  protocols (more than just HTTP(S): e.g., WebSockets or gRPC.
* Authentication

An ingress controller is a DaemonSet or Deployment, deployed as a Kubernetes Pod, that watches the endpoint of the API server for updates to the Ingress resource. Its job is to satisfy requests for Ingresses.

## Why does Ingress require a load balancer
Ingress is tightly integrated into Kubernetes, meaning that your existing workflows around `kubectl` will likely extend nicely to managing ingress. An Ingress controller does not typically eliminate the need for an external load balancer , it simply adds an additional layer of routing and control behind the load balancer.
Pods and nodes are not guaranteed to live for the whole lifetime that the user intends: pods are ephemeral and vulnerable to kill signals from Kubernetes during occasions such as:
* Scaling.
* Memory or CPU consumption
* Downtime due to outside factors.
The load balancer (Kubernetes service) is a construct that stands as a single, fixed-service endpoint for a given set of pods or worker nodes. To take advantage of the benefits of a Network Load Balancer (NLB), we create a Kubernetes service of `type:loadbalancer` with the NLB annotations, and this load balancer sits in front of the ingress controller – which is itself a pod or a set of pods. In AWS, for a set of EC2 compute instances managed by an Autoscaling Group, there should be a load balancer that acts as a load balancing mechanism.

![](index/11tDfN7IqiC8qv7bZS50qw%202.png)

The diagram above shows a Network Load Balancer in front of the Ingress resource. This load balancer will route traffic to a Kubernetes service (or Ingress) on your cluster that will perform service-specific routing. NLB with the Ingress definition provides the benefits of both a NLB and an Ingress resource.

## Install ingress controller 
Install the `nginx` ingress controller
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/aws/deploy.yaml
```

Confirm the ingress controller was installed correctly and is running
```sh
kubectl get pods -n ingress-nginx
```

Output: 
```
NAME                                      READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-2f6hh      0/1     Completed   0          31h
ingress-nginx-admission-patch-b9wjz       0/1     Completed   0          31h
ingress-nginx-controller-54bfb9bb-zrv2v   1/1     Running     0          31h
```

## Create services
Create two services (`apple.yaml` and `banana.yaml`) to demonstrate how the Ingress routes a request.  Run two web applications that each output a slightly different response. Each of the files below has a service definition and a pod definition.

```sh
kubectl apply -f https://raw.githubusercontent.com/cornellanthony/nlb-nginxIngress-eks/master/apple.yaml 

kubectl apply -f https://raw.githubusercontent.com/cornellanthony/nlb-nginxIngress-eks/master/banana.yaml
```

## Create ingress rules
Now declare an Ingress to route requests to `/apple` to the first service, and requests to `/banana` to second service. 

Review the Ingress manifest below. Pay attention to the `rules` field that declares how requests are passed along:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: example.com
    http:
      paths:
        - path: /apple
          pathType: Prefix
          backend:
            service:
              name: apple-service
              port:
                number: 5678
        - path: /banana
          pathType: Prefix
          backend:
            service:
              name: banana-service
              port:
                number: 5678
```

Create a new file `nginx-ingress.yaml` with the rules from above and apply it. 

Confirm the ingress has been created: 
```sh
kubectl get ingress 
```

Output: 
```
NAME              CLASS    HOSTS         ADDRESS                                                                         PORTS   AGE
example-ingress   <none>   example.com   a5811c37561314f62ae0b1c26c755b43-fddf55b7c7350d8b.elb.us-west-2.amazonaws.com   80      3m49s
```

The ingress will route requests to `/apple` to the `apple` service, and `/banana` to the banana service. 

## Test the ingress rules
To confirm this we need to curl the Ingress. 

Set an environment variable: 
```sh
export INGRESS_IP=$(kubectl get ingress example-ingress --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
``` 

Query the endpoints
### Apple
```sh
curl -H "Host: example.com" http://$INGRESS_IP/apple
```

Output: 
```
apple
```

### Banana
```sh
curl -H "Host: example.com" http://$INGRESS_IP/banana
```

Output
```
banana
```

There is much more you can do with Kubernetes Ingress rules. Read about everything in the [documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)

## Cleanup
```sh
kubectl delete -f nginx-ingress.yaml
kubectl delete pod apple-app banana-app
kubectl delete svc apple-service banana-service
kubectl delete ns ingress-nginx
```
