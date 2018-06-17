---
published: false
---
## JVM and Docker memory limits


In my first blog post I decided to talk a little bit about running JVM based applications in Docker containers, specifically about the memory gotchas that you will run into.

If you were to run an application(on JRE < 10) in a container without specifying the heap size limit with an `-Xmx` flag(note that this option alone is not sufficient for production, see [here](https://medium.com/@matt_rasband/dockerizing-a-spring-boot-application-6ec9b9b41faf)) the application might unexpectedly crash with an `OutOfMemoryError` exception. 

The reason for that is twofold:
- The memory allocated to the container is lower than the host's available RAM(by default, Docker will use all available memory, you can look it up by running `docker stats`)
- The inability of the JVM to detect the fact that it's running in a containerized environment.

I've had these crashes happen to me at work. We are using Amazon ECS for container orchestration, and as part of the ECS configuration we need to set hard CPU/memory limits for each container(which are called "tasks" in Amazon parlance). Due to human error, these limits were set lower than the value of the `-Xmx` flag, causing the JVM to allocate more memory than was available, which led to the docker daemon killing the 'misbehaving' container.

After a couple of incidents like that, I decided to do some googling on the subject. Turns out there's an experimental flag available that makes the JVM aware of the fact that it's running in a container(on Linux Docker uses control groups or [cgroups](https://sysadmincasts.com/episodes/14-introduction-to-linux-control-groups-cgroups) kernel feature to control resources for its containers). Actually, It's  2 options. Here they are: `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`. If these are passed in and `Xmx` is not specified, the JVM will calculate the max heap size according to this formula: `Xmx = MaxRAM / MaxRAMFraction`. The default value for `-XX:MaxRAMFraction` is 4, so by default the heap will be 25% of available RAM, which is far from ideal. You might want to set it to 1. it would be a bad idea, however--there's also non-heap memory that you need to account for, not to mention the additional RAM the container might need(`docker exec` comes to mind). 
