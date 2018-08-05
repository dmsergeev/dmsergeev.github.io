---
published: true
title: A Problem With The Shell Execution Form in Docker
---
Look at the Dockerfiles for your applications. Do you see `ENTRYPOINT` or `CMD` instructions in the following format: `ENTRYPOINT node app.js` or `ENTRYPOINT [""sh"", ""-c"", ""node app.js""]`? If yes, then you might have a problem.

The other day I had a realization that I had never seen our containerized applications gracefully shut down. You would usually expect to see at least a couple of log messages about closing the application context(we use Spring) on a `SIGTERM` signal(it is a termination signal politely asking the application to shut down, as opposed to `SIGKILL` which is an immediate termination). I checked Kibana--nothing. I ssh'ed into an ECS EC2 instance to check the logs and, again, saw nothing. The applications seem to had been instantly killed, which didn't really make sense. Or did it?

I decided to double check and started one of services locally circumventing Docker, then I ran `kill` which sends a `SIGTERM` signal to the application. Looked at the logs and there they were:

`[INFO] Stopping service [Tomcat]`

`[INFO] Stopping ProtocolHandler ["http-nio-9104"]`

`[INFO] Destroying ProtocolHandler ["http-nio-9104"]`

`[INFO] Shutting down the Executor Pool for PollingServerListUpdate`




