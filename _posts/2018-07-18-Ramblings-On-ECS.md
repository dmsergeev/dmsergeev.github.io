---
published: false
title: Ramblings On ECS
---
After working with ECS for quite some time I decided to talk a little bit about my experience with it and compare it to Kubernetes.

First, a little bit of background. AWS ECS is a managed container orchestration service with which you can deploy and run your dockerized applications. It supports various types of 'workloads':

- **Task**, this is basically your application. Can run multiple containers in 1 task. Think of them as Pods in k8s.
- **Scheduled task**, runs tasks on a user defined schedule. In k8s that would be a CronJob resource.
- **Service**, a self-healing collection of tasks. Allows you to specify the number of instances of a task, auto-scaling policy, and deployment strategy. It also interacts with your elastic load balancers to register/deregister targets. Even though this sounds a like a lot, it lacks some features of Deployment resources in k8s(deployment pause and rollback).

An important note to make here is that there's no way to run ECS workloads on anything but EC2(either managed by you, or by using Fargate, which I will discuss later), meaning that once you start using it, you are pretty much locked into using AWS, whereas with k8s you can run your infrastracture anywhere. This is probably my biggest gripe with ECS, and to be honest, with most of AWS services in general. They are tightly coupling their offerings, which is great for them, but not the consumer. There's an upside to this, however. Since everything is so coupled you can easily take advantage of other services with almost no configuration(ELB, CloudWatch).

At work we are running approximately 50 services in multiple availability zones across 5 clusters(Namespaces in k8s), most of the time it works fine... except when it doesn't.










