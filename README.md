# compose-executor

compose-executor project aims to enable Mesos frameworks to launch a pod of docker containers. Kubernetes/K8 introduced the notion of collection of docker containers that shares namespaces and treated the collection as a single scaling unit. Brendan Burns talked about some design patterns/use cases for pods in [DockerCon'15](https://www.youtube.com/watch?v=Ph3t8jIt894).

Docker Compose is a cherished tool used in docker community that helps us model a collection of docker containers. The specification is very flexible where the containers can be launched in multi hosts, or a single host and then you can also model a pod collapsing namespaces (net, IPC supported in docker, PID support coming). The spec is also gaining popularity by 3rd party platforms/IDE like Azure (baked in Visual Studio support), AWS also added support behind it since DockerCon'15.

## Goal

The project goal is to model a pod of containers with docker-compose and launch it with your favorite Mesos frameworks like Marathon, Apache Aurora, etc. One does not need to switch to Kubernetes on Mesos if all that they are looking is to launch pods (model pod like workloads). With network and storage plugins supported directly in Docker, one can model advanced pods supported through compose. Also instead of using some different spec to define pods, we wanted to build around compose that is well accepted in the docker community.

## Tasks
1. Write a general purpose docker-compose executor so that it can be used by any Mesos frameworks (We are delivering a patch in Apache Aurora to have custom executor support other than thermos. Marathon can readily support it). Executor will launch a pod of containers per task and maintain the lifecycle.
2. DockerContainerizer is not best suited to run it because it can launch only one container per task and has few other shortcomings. So, we plan to use MesosContainerizer with cgroups isolator and then with the newly supported cgroups-parent flag, put the docker containers in the pod under the executor cgroup so that it’s a hierarchy. We are delivering a patch in docker-py and compose to support this docker flag.
3. Using Mesos hooks, we plan to catch rare conditions where executor crashes and containers can leak and then do cleanup of those.
4. The compose spec file can be downloaded in the mesos sandbox through mesos task URL.
5. We will make sure the container names haves proper prefix to reflect the mesos task.
6. Multiple pods can be launched in the single host without conflicts.
7. Container stdout/stderr logs should be redirected to sandbox stdout/stderr.
8. Going to write a  standalone tool that can parse the compose file to add up the resources (cpu/mem) of the containers in the compose spec because for Mesos it’s a single task. This is needed before submitting the task to Marathon/Aurora. Surely in future there can be frameworks which consumes the compose file in the framework and handles it in server side with local pod or distributed containers treated a unit (compose with docker network namespace say) and spread across multiple tasks. We are not modelling that for now.
9. We are going to write a sample compose file trying to model like a pod collapsing net namespace. Once pid namespace is collapsed, one can run tools like wetty in a separate container in the pod and give terminal emulator to the pod (avoid ssh/docker exec access). We will show a wetty integration merged into a container rather than a separate container for now. Other monitoring/debug tools can be launched in a separate container in the pod.

## Example compose file

This is a sample compose file modelling a pod (collocated set of containers treated as a unit):
```yaml
baseC:
  container_name: baseC
  image: centurylink/wetty-cli
  net: "bridge"
  ports:
   - "5000:5000"
   - "8000:3000"
 
web:
  container_name: web
  build: .
  net: "container:baseC"
  volumes:
   - .:/code
  links:
   - redis
redis:
  container_name: redis
  image: redis
  net: "container:baseC"
```

In real life use case, as Brendan also explained in the talks the pod will comprise of main app container with side car containers such as amsaddor/proxy containers, log aggregators, monitoring/debug containers etc. For sake of simplicity and common example, we are putting both web and redis in the same pod. Useful for development clusters and demo :)
 
In the compose file above, baseC is the primary container getting the IP assigned from docker bridge and is also responsible for advertising ports for other containers in the pod (docker does not allow for other containers joining the network namespace expose ports at that point. Hence, primary container exposing all the ports and secondary containers in the pod will bind to those).

We are also running wetty (https://github.com/krishnasrinivas/wetty ) inside the primary baseC container, to have terminal emulator to access the pods. Port 5000 is app port, 8000 is wetty port, redis port not exposed outside pod.
 
The redis and web container share the network namespace of the baseC container.
 
Here is the output of running netstat –plant from wetty terminal (ADD a screen shot here with netstat output):
```
TBD: Tejas
```
 
This shows information about all the bind ports in different containers in pod.
 
We cannot share pid namespace yet and that is coming soon
https://github.com/docker/docker/issues/10163
 
With Wetty one can avoid ssh/docker exec into the pod and use the browser terminal emulation.
 
Instead of static host port mapping, we can leverage dynamic ports as mesos resource and map it in compose file. However, if the network model assigns a routable IP (instead of private host ip’s, default bridge model), we don’t need to expose host ports as such.

## Licensing
compose-executor is licensed under the Apache License, Version 2.0. See [LICENSE](/LICENSE) for the full license text.
