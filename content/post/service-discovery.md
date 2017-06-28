+++
Description = "Service Discovery with Kuberenes"
Tags = ["Development", "sd"]
Categories = ["Development", "kubernetes"]
menu = "main"
date = "2017-06-24T16:08:07+01:00"
title = "Service Discovery with Kuberenes"
+++

How to do **Service Discovery** with Kubernetes is one of those questions that come up over and over during the trainings. Even seasoned kubernetes users struggle sometimes with this concept, so, I've decided to write a bit to make it clearer.

What is Service Discovery? According to [Wikipedia](https://en.wikipedia.org/wiki/Service_discovery):

> "Service discovery is the automatic detection of devices and services offered by these devices on a computer network."

In Kubernetes we have the concept of [Services](https://kubernetes.io/docs/concepts/services-networking/service/), which is an abstraction that defines a logical set of Pods determined by a Label Selector.

This is, you can label a set of Pods and access them using a service with that label as selector.

As we know, Pods will come and go, but the service accessing a set of pods idetified by the pair label/selector will remain constant. This provides different possibilities. We can use that service or create our own.