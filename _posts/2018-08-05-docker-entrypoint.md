---
published: true
title: A Problem with the Shell Execution Form in Docker
---
Look at the Dockerfiles for your applications. Do you see `ENTRYPOINT` or `CMD` instructions in the following format: `ENTRYPOINT node app.js` or `ENTRYPOINT ["sh", "-c"", "node app.js"]`? If yes, then you have a problem.

The other day I had a realization that I had never seen our containerized applications gracefully shut down. You would usually expect to see at least a couple of log messages about closing the application context(we use Spring) on a `SIGTERM` signal(it is a termination signal politely asking the application to shut down, as opposed to `SIGKILL` which is an immediate termination). I checked Kibana--nothing. I ssh'ed into an ECS EC2 instance to check the logs and, again, saw nothing. The applications were being instantly killed.

This is a big problem. Without a graceful shutdown, all in-flight HTTP requests are dropped. The message consumers might not finish processing a batch of messages. The database connections will linger. The list goes on. We haven't actually noticed any problems just because most of our services expose HTTP endpoints and we do rolling updates to ensure no downtime--the system removes the application from a load balancer and then drains connections for a while before stopping it. Also, up until very recently, we did not have high throughput message consumers/producers. I was very lucky to catch this before we ramped up the volume on these services.

I decided to verify the theory and started one of the services locally circumventing Docker, then I ran `kill` which sends a `SIGTERM` signal to the application, looked at the logs and there they were, among others:

>[INFO] Stopping service [Tomcat]
>[INFO] Stopping ProtocolHandler ["http-nio-9104"]
>[INFO] Destroying ProtocolHandler ["http-nio-9104"]

For some reason, a dockerized application was ignoring `SIGTERM`. I ran `docker exec CONTAINER_ID ps -o "pid ppid command"` to see the process tree:
>[ec2-user@host ~]$ ps -o "pid ppid command"
>  PID  PPID     COMMAND
>    6     1   java -jar
>    1     0   sh -c  

Turns out our application is, actually, a child of the shell process. The problem is that **the shell does not proxy termination signals down to its children**.

The `ENTRYPOINT` instruction in all of our Dockerfiles was in the shell form[1], as opposed to exec form that does not spawn a shell process. It looked like this:
`ENTRYPOINT ["sh", "-c", "java -jar /app.jar "]`. Changing it to `ENTRYPOINT ["java", "-jar /app.jar"]` fixed the issue and I started to see shutdown logs in Kibana. I am yet to find a real-life use case for the shell form... Apart from a need to pass in environment variables? Not sure why it would be needed, though, when you could look them up in the code in most runtimes.

You can also accidentally be using a shell form and not knowing about it. For some reason, Docker engineers decided to make `ENTRYPOINT node app.js` a shell form, so you'd have a shell process even though it was not explicitly requested(like with `ENTRYPOINT ["sh", "-c"]`). I really don't like when software that does that. Very confusing.

**You should avoid the shell form and always use the exec form of the entrypoint/cmd instruction.**


[1] https://docs.docker.com/engine/reference/builder/#entrypoint