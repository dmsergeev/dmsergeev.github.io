---
published: true
title: A Problem with the Shell Execution Form in Docker
---
Look at the Dockerfiles for your applications. Do you see `ENTRYPOINT` or `CMD` instructions in the following format: `ENTRYPOINT node app.js` or `ENTRYPOINT ["sh", "-c"", "node app.js"]`? If yes, then you might have a problem.

The other day I had a realization that I had never seen our containerized applications gracefully shut down. You would usually expect to see at least a couple of log messages about closing the application context(we use Spring) on a `SIGTERM` signal(it is a termination signal politely asking the application to shut down, as opposed to `SIGKILL` which is an immediate termination). I checked Kibana--nothing. I ssh'ed into an ECS EC2 instance to check the logs and, again, saw nothing. The applications were being instantly killed.

This is a big problem. Without graceful shutdown all in-flight HTTP requests are dropped. The message consumers might not finish processing a batch of messages. The list goes on. We haven't actually noticed any problems just because most of our services expose HTTP endpoints and we do rolling updates to ensure no downtime--the system removes the application from  a load balancer and then drains connections for awhile before stopping it. Also, up until very recently we did not have high throughput Kafka consumers/producers. I was very lucky to catch this before we ramped up the volume on these services.

I decided to double check and started one of services locally circumventing Docker, then I ran `kill` which sends a `SIGTERM` signal to the application. Looked at the logs and there they were:

`[INFO] Stopping service [Tomcat]`

`[INFO] Stopping ProtocolHandler ["http-nio-9104"]`

`[INFO] Destroying ProtocolHandler ["http-nio-9104"]`


The evidence pointed to Docker. For some reason, the application was oblivious to `SIGTERM`. Naturally, I turned my look to the Dockerfile. 




