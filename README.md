# Kubernetes Ingress Guide

## Table of Contents

- [Chapter 1 - Introduction to Kubernetes Ingress](#chapter-1---introduction-to-kubernetes-ingress)
- [Chapter 2 - Ingress Resources](#chapter-2---ingress-resources)
- [Chapter 3 - TLS and HTTPS using Ingress](#chapter-3---tls-and-https-using-ingress) 
- [Chapter 4 - Advanced Ingress](#chapter-4---advanced-ingress)
- [Chapter 5 - Ingress Monitoring and Troubleshooting](#chapter-5---ingress-monitoring-and-troubleshooting)
- [Chapter 6 - Migrating to Ingress](#chapter-6---migrating-to-ingress)
- [Chapter 7 - Conclusion](#chapter-7---conclusion)

## Introduction

Welcome to my in-depth guide on Kubernetes Ingress! 

Ingress is one of the most powerful features Kubernetes offers for managing external access to your cluster services. If you've been struggling with things like scaling multiple load balancers or routing traffic to different backend apps, Ingress is the solution you need.

In this comprehensive guide, I'll walk you through everything you need to know about Ingress, from basic concepts to advanced configuration and troubleshooting. I aim to provide plenty of real-world examples and sample YAML so you can easily follow along. 

Whether you're completely new to Ingress or looking to level up your skills, you'll find this guide helpful. My goal is for you to finish reading with a solid understanding of:

- The role of Ingress controllers like NGINX for handling external traffic
- How to create Ingress resources to route requests to backend services
- Securing your apps with TLS certificates using Cert-Manager
- Advanced routing techniques like path, host and header based rules
- Monitoring and troubleshooting your ingress deployment
- Strategies for migrating from other ingress solutions

I'll share tips and gotchas I've learned from implementing Ingress across many Kubernetes clusters over the years. This guide is written in plain language, so no need to be a Kubernetes expert to follow along!

Ready to master the ins and outs of Kubernetes Ingress? Let's get started! The first chapter covers the basics of what Ingress is and why it's so useful...

## Chapter 1 - What is Kubernetes Ingress?

Ingress is one of those Kubernetes things that makes you go "Aha, now I get it!". It solves a key problem in a Kubernetes native way. 

The problem is simple - you have deployed multiple app services in your Kubernetes cluster and you want to expose these services to external users. Examples of external users are end users of your web apps or mobile apps, as well as external clients calling your API services.

The default Kubernetes way to expose a service externally is by using a Service of type LoadBalancer. This provisions a cloud load balancer for your service. But there are limits to this approach:

- Load balancers are expensive - you have to pay for every one provisioned.
- Wildcard domains and path based routing are not easy to setup. 
- SSL termination is complex to manage on load balancers.

This is where Ingress comes to the rescue! Ingress provides a Kubernetes native way to manage external access to your cluster services. 

Here are some key things Ingress gives you:

- Consolidated external access - You can route all external traffic through a single IP address. This cuts down on number of load balancers needed. 

- Host and path based routing - Easily route traffic to services based on hostname or URL path. Useful for configuring multiple services on same IP.

- Centralized SSL/TLS termination - Terminate SSL at the ingress and serve services over HTTP internally. Removes overhead of SSL management from your services.

- Simple to set up URL rewriting and redirects

- Works out of the box on all major cloud providers 

- Lots of flexibility to customize via annotations

To deploy an ingress controller on my DigitalOcean Kubernetes cluster, I first created a namespace for it:

```
kubectl create namespace ingress
```

Then I deployed the popular NGINX ingress controller in this namespace:

``` 
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/do/deploy.yaml -n ingress
```

This brought up an NGINX ingress controller pod and service in the cluster. The ingress controller automatically provisions a load balancer to expose itself. 

I could now create Ingress resources to route traffic from this controller to my services! More on that in the next chapter.

So in summary - Ingress gives you easy, flexible and native way to expose your Kubernetes services to external traffic. It avoids many limitations with just using load balancers directly.

## Chapter 2 - Ingress Resources

In the last chapter we set up an NGINX ingress controller to handle external traffic to the cluster. Now let's shift gears and see how to use Ingress resources to route requests to our services.

An Ingress resource is a Kubernetes object that defines rules to route external requests to services running in the cluster. Here's a simple example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress  
spec:
  rules:
  - host: foo.mydomain.com 
    http:
      paths:
      - pathType: Prefix
        path: /bar
        backend:
          service: 
            name: my-service
            port: 80
```

This Ingress routes requests from host foo.mydomain.com/bar to my-service on port 80.

Some key things about Ingress resources:

- Uses host and path based routing to services
- Each http rule maps a URL path to a backend service
- Supports name based virtual hosting
- Backend refers to a Kubernetes service + port

To test this out, I already had a simple hello-world app deployed:

```
kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0  
kubectl expose deployment web --port=8080 --target-port=8080 --name=hello
```

This runs the hello app on port 8080. I created the above Ingress and confirmed I could access the app on foo.mydomain.com/bar.  

You can configure path based routing to multiple backends on same host, TLS, whitelisting and more. We'll cover those in later chapters. 

With Ingress resources, you get a very flexible way to expose multiple services on same IP address and configure advanced edge routing. Definitely take advantage of it instead of creating multiple load balancers!

So that covers basic Ingress resources. We created a simple one to route requests from a hostname/path to an backend service. Next up we'll look at deploying TLS and HTTPS for our Ingress.

## Chapter 3 - TLS and HTTPS using Ingress 

So far we have configured HTTP routing to services using Ingress. While HTTP is fine for internal traffic, external traffic should always use HTTPS and TLS for security.

In this chapter, we will set up TLS termination and HTTPS for an Ingress host using a TLS certificate. 

There are several ways to configure TLS certificates and keys for Ingress. For my guide, I will use CertManager which integrates natively with Kubernetes.

First, I installed CertManager into its own namespace:

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

This brings up cert-manager pods and services. CertManager can do TLS automation across various Kubernetes integrations.

Next, I created an Issuer resource to generate Let's Encrypt TLS certificates automatically:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt   
spec:
  acme: 
    server: https://acme-v02.api.letsencrypt.org/directory
    email: myemail@mydomain.com
    privateKeySecretRef:
      name: letsencrypt-private-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

This will automatically request TLS certificates from Let's Encrypt ACME server when I create Certificate resources.

Now I modified my earlier Ingress with a TLS section: 

```yaml
apiVersion: networking.k8s.io/v1  
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  name: my-ingress   
spec:
  tls:
  - hosts:    
    - foo.mydomain.com
    secretName: my-tls-secret
  rules:
  - host: foo.mydomain.com  
    http:
      paths:
      - path: /
        backend:
          serviceName: hello
          servicePort: 8080
```

This instructs CertManager to get a TLS certificate for my host and store it in secret my-tls-secret. 

I could now access my Ingress at https://foo.mydomain.com securely! The TLS offload happens at the ingress controller.

This way you can easily add HTTPS support for your apps via Ingress TLS configuration. CertManager handles the complex parts like issuing and renewing certificates automatically.

Up next we'll look at some advanced Ingress capabilities like header based routing, access control and rate limiting.

## Chapter 4 - Advanced Ingress

So far we've covered the basics of traffic routing and TLS with Ingress. Now let's look at some advanced capabilities that make Ingress incredibly powerful.

### Header based routing

Ingress allows routing traffic based on headers as well as host/path. For example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - path: /foo  
        pathType: Prefix
        backend:
          service:
            name: foo-service
            port: 80
        headers:
          testheader: foo
      - path: /bar
        pathType: Prefix 
        backend:
          service:
            name: bar-service
            port: 80
        headers:
          testheader: bar
```

This will send requests with testheader: foo to foo-service, and testheader: bar to bar-service. 

Header based routing allows you to split traffic dynamically across multiple backends.

### Access control 

Ingress access can be locked down to allow only specific source IPs or IP ranges via the ingress.kubernetes.io/whitelist-source-range annotation:

```yaml
metadata:
  annotations: 
    ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/24, 172.16.0.0/16"
```

This will only allow access from IPs in the specified ranges.

### Rate limiting

The nginx.ingress.kubernetes.io/limit-rps annotation can be used to rate limit traffic to a particular service:

```yaml 
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
```

This will limit requests per second to 5. Useful for protecting against attacks or burst traffic.

These are just a sample of powerful capabilities you get with Ingress for managing external access. In later chapters we'll cover monitoring, debugging and more. 

I hope this overview gives you a sense of what's possible with advanced ingress configuration!


## Chapter 5 - Ingress Monitoring and Troubleshooting

Now that we have set up Ingress resources and routing, let's look at how to monitor traffic and troubleshoot issues.

### Monitoring Ingress Traffic

It's useful to have visibility into metrics like requests, errors and latency for ingress. The NGINX ingress controller exposes a metrics endpoint at /metrics by default. 

We can collect these metrics in Prometheus by creating a ServiceMonitor object:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor  
metadata:
  name: ingress-nginx
  namespace: monitoring
spec:
  endpoints:
  - port: metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - ingress
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```

This will scrape /metrics from all ingress controllers and collect metrics in Prometheus. We can then graph metrics like ingress_requests_total, ingress_controller_success and ingress_controller_requests.

These charts help spot traffic spikes, errors and saturation issues with the ingress.

### Access Logging 

Ingress controllers can also log details about each request they handle. For NGINX, access logs are enabled using an annotation:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress 
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-access-log: "true"   
```

The logs get stored in JSON format and can be parsed to analyze traffic patterns, IPs, user agents etc.

### Debugging Issues

If you spot errors or unexpected traffic drops in monitoring, some things to check are:

- Incorrect ingress host, path or service configuration
- Backend service downtime or high latency  
- DNS resolution failures from external clients
- 403 errors due to IP whitelisting issues

The ingress controller and backend logs can provide helpful debugging details. Also check custom metrics exposed by the controller.

This covers the basics of monitoring and troubleshooting your ingress deployment. With these practices you can gain visibility and rapidly detect issues with external access.

## Chapter 6 - Migrating to Ingress 

For apps running on Kubernetes, there are often requirements to migrate external traffic routing to Ingress from other solutions like load balancers. 

In this chapter we'll go over patterns for migrating existing apps to use Ingress for traffic management.

### Migrating from LoadBalancers

Many apps use LoadBalancer type services to expose them externally. To migrate to Ingress:

1. Deploy an ingress controller if not already present.

2. Create an Ingress resource to route traffic to the existing LoadBalancer service.

For example:

```yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress  
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-lb-service
            port: 
              number: 80
```

3. Configure external traffic to hit the Ingress IP instead of LoadBalancer IP.

4. Once traffic is routed via Ingress, the LoadBalancer service can be deleted.

This allows incrementally migrating apps using LoadBalancers over to an Ingress based routing model.


### Migrating from other API Gateways

For apps using API Gateways or custom proxies, Ingress can replace those with Kubernetes native routing.  

The process is similar:

1. Deploy Ingress controller
2. Create Ingress for app with routing rules
3. Point external traffic to Ingress IP 
4. Remove old gateway/proxy


### Conclusion

Migrating apps to Ingress helps consolidate external access and gain consistent routing, security and visibility. For HTTP based workloads, it's ideal to use Ingress over custom proxies or gateways.


## Chapter 7 - Conclusion

We have reached the end of our in-depth guide on Kubernetes Ingress. Let's recap what we have covered:

- Introduction to Ingress and its benefits for external access

- Deploying a simple Ingress controller on Kubernetes

- Creating Ingress resources to route traffic to backend services

- Configuring TLS termination and HTTPS using CertManager

- Advanced Ingress capabilities like header routing, access control and rate limiting

- Monitoring and troubleshooting your Ingress deployment

- Strategies for migrating existing apps to use Ingress


We went hands-on and set up a full-fledged Ingress environment, secured it with TLS, routed traffic to example apps, and saw how to monitor logs and metrics.

I hope you now feel equipped to implement Ingress for your own Kubernetes applications and services. The key takeaways are:

- Use Ingress for simplified external access, instead of multiple LoadBalancers  

- Ingress resources allow flexible, host and path based routing

- Advanced features like header routing, access control provide edge control

- TLS termination removes overhead of certificate management from apps

- Proper monitoring and debugging practices are must for reliability

- Ingress can incrementally replace other ingress solutions like API gateways

Kubernetes documentation has more detailed reference on Ingress capabilities. Feel free to reach out if you have any other questions!
