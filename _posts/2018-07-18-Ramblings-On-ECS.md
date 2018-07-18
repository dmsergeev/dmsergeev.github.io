---
published: false
title: Ramblings On ECS
---
After working with ECS for quite some time I decided to talk a little bit about my experience with it and compare it to Kubernetes.

First, a little bit of background. AWS ECS is a managed container orchestration service with which you can deploy and run your dockerized applications. It supports various types of 'workloads':

- **Task**, this is basically your application. Can run multiple containers in 1 task. Think of them as Pods in k8s.
- **Scheduled task**, runs tasks on a user defined schedule. In k8s that would be a CronJob resource.
- **Service**, a self-healing collection of tasks. Allows you to specify the number of instances of a task, auto-scaling policy, and deployment strategy. It also interacts with your elastic load balancers to register/deregister targets. Even though this sounds a like a lot, it lacks some features of Deployment resources in k8s.

An important note to make here is that there's no way to run ECS workloads on anything but EC2(either managed by you, or by using Fargate[1]), meaning that once you start using it, you are pretty much locked into using AWS, whereas with k8s you can run your infrastracture anywhere. This is probably my biggest gripe with ECS, and to be honest, with most of AWS services in general. They are tightly coupling their offerings, which is great for them, but not the consumer. There's an upside to this, however. Since everything is so coupled you can easily take advantage of other services with almost no configuration(ELB, CloudWatch).


At work we are running  ~50 services in multiple availability zones for each environment. Most of the time it works fine... except when it doesn't.

2 major problems I can group under a **Lack of Visibility** umbrella:
- If your task abruptly exited with a non-zero code there's no way to see why. The ECS event log console is empty. All information you can get is an exit code and endless task restarts if it was a part of a service. To actually troubleshoot this, you'd have to ssh into the EC2 instance and examine the logs yourself. We could probably improve this by using CloudWatch log(the UI to view the logs definetely needs improvements) driver. Compare it to `kubectl logs`...
- If your cluster has no capacity to place the task, ECS events log will say just that and... that's it. Your deployment is basically stalled, and you wouldn't even know that. Even after you add a EC2 node to the cluster, there's no information available(I couldn't find anything) on when it's going to try again. Sometimes it's 15 minutes, sometimes it's over an hour(!). The workaround here is to trigger it manually. Not good.

Other problems I've experienced with it:
- ECS agent randomly dying or not doing anything requiring us to kill the node.
- A task that ECS reported was removed from the target group was still receiving connections, this might've been an ELB issue, haven't figured it out. Support seems to have forgotten about my ticket.


All in all, I see no reason for somebody who's choosing an orchestration system to go with ECS. It just can't compete with Kubernetes in terms of features(this will probably need another post) and, now, ease of operation with a lot of cloud providers offering managed clusters. Not to mention that k8s is fully open-source with thousands of people contributing, whereas ECS not only proprietary, but also locks you into using AWS.


[1] Fargate is a service that allows you to run containers without managing the EC2 instances. It integrates tightly with ECS and works pretty well, in my experience. One problem with it is that if you run your containers with it you **cannot** use private image registries apart from ECR.
