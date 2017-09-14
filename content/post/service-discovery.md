+++
Description = "Service Discovery with Kubernetes"
Tags = ["Development", "sd", "services", "istio", "labels", "service discovery"]
Categories = ["kubernetes", "architecture"]
menu = "main"
date = "2017-09-14T11:08:07+01:00"
title = "Service Discovery with Kubernetes"
+++

How do you do **Service Discovery** in Kubernetes? 

This is one of those questions that come up over and over during the trainings. Even seasoned kubernetes users sometimes struggle with how service discovery works.

Let's start from the beginning: What is Service Discovery? According to [Wikipedia](https://en.wikipedia.org/wiki/Service_discovery):

> "Service discovery is the automatic detection of devices and services offered by these devices on a computer network."

In Kubernetes we have the concept of [Services](https://kubernetes.io/docs/concepts/services-networking/service/), which is an abstraction that defines a logical set of Pods determined by a Label Selector.

This means that you can label a set of Pods and access them using a service with that label as a selector.

As we know, Pods will come and go, that's why we need a service, that provides a fixed IP and DNS. There's a [excellent](https://skillsmatter.com/skillscasts/10466-deep-dive-on-kubernetes-networking) talk by [Tim Hockin](https://twitter.com/thockin) where he explains how the networking works, in case you're interested in knowing more.

Let's go back to our pods/services. We have seen that you can label a pod or set of pods. Pods created by `Deployments` or `ReplicaSets` will share a set of labels, but nothing is stopping you from adding more labels. In fact, using labels show the level of maturity on using Kubernetes. Why? Because labels are the way of identifying and grouping resources in kubernetes.

To label a resource you can use the `kubectl label` instruction:

```
kubectl label MYPOD build=snapshot
```

This will allow you to create a service that accesses all the pods that have the label `build: snapshot`.

There's something worth noting. A service is not a real Load Balancer. A service is an abstraction. The service IP will be used in the IPTABLES rules to redirect traffic to the right pod (IP). This means that there's no real endpoint.

Let's look at an example. Imagine you're writing an application that has to call a microservice called `Suggestions`, this service will provide all the suggestions related to the object you specify. The team responsible for `Suggestions` has created a service: `suggestions.default.svc.cluster.local`.

This means that our application just needs to call `suggestions.default.svc.cluster.local` to get suggestions back. For example, we can create an env var `SUGGESTIONS_URL` and set the value to `suggestions.default.svc.cluster.local`.

This approach works, and it's very common to find in many kubernetes platforms. Do you see any problem with this approach? Think about it, does it not feel familiar? Yes, it does, because service discovery was created to solve the issue that this approach faces: changes in the dns of a dependency will break our application.

Service discovery is, as we said above, `the automatic detection` which is precissely what we don't have if we hardcode a URL/service in our application. How can we solve this problem? We know kubernetes provides service discovery, so, how can we use it?

Let's look at what we have so far:

* Suggestions application
* Suggestion service pointing to pods running our suggestions application
* An application that consumes Suggestions calling that service.

What if, we create our own suggestions service? What if our application defines which DNS/URL wants to use and delegates on service discovery and Kubernetes to provide the endpoints? Bear in mind that we're just creating a new IPTABLES rule (no new physical resources), but now we control the DNS that our application uses and we decide which pods are the right candidates to become part of that service and kubernetes handles the endpoints.

Why do we want to do this? Does it not sound like overkill? It might. We all agree that this approach doesn't work for extreme use cases: imagine an application consumed by hundreds of applications. Does it make sense to create a service per application dependency? Probably not, but in the majority of cases if you have such kind of applications they include their own way of Load Balancing, but for the majority of applications, this approach works way better because it provides more resilience, more control over things that can break your application.

We always recommend to approach managing applications in the same way:

* Define your dependencies and create a ReadinessProbe that checks that all the core dependencies are ok. By core dependencies, we mean the ones that without them our application cannot work properly.
* Define your own services to access those dependencies.
* Ideally you will wrap up all the application resources in a logical release, that helm provides by default.

Now, when you deploy your application you're not only creating your runtime, you're creating your runtime and all the kubernetes services that connect to your application dependencies. Those resources are under your control, not under the team in charge of those applications.

It might seem a small detail, but if you control the services that get in and out of your application, you control the routing.

Platforms like [Istio](https://istio.io), the service mesh, use the kubernetes service discovery (labels and selectors) to redefine how applications access resources. Their approach is way more exhaustive that the one here described providing even finer grained control over the network functions of the applications. However, it's a bit early for Istio but we're very excited about the direction and the features this platform provides.