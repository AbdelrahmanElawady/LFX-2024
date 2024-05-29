![CNCF - KubeArmor Logo](https://raw.githubusercontent.com/kubearmor/KubeArmor/main/.gitbook/assets/logo.png)

**Organization: [CNCF](https://www.cncf.io/)**

**CNCF Project: [KubeArmor](https://kubearmor.io/)**

**Project: [Leverage OCI Hooks for Container Events](https://mentorship.lfx.linuxfoundation.org/project/70f1ef34-7ddd-466d-a5bc-0f74e98c06f8)**

**Mentors: [Barun Acharya](https://github.com/daemon1024), [Rudraksh Pareek](https://github.com/DelusionalOptimist), [Akshay Gaikwad](https://github.com/akshay196)**

## Motivation

KubeArmor is a cloud-native runtime security enforcement system that restricts the behavior of pods, containers, and nodes (VMs) at the system level. It utilizes container runtime UNIX sockets to directly monitor containers.
This polling model has some downsides due to mounting runtime socket directly inside KubeArmor container and latency between containers starting and containers being monitored.
The goal was to change KubeArmor model to overcome these downsides.

## Project Goals

Since KubeArmor depends on container runtime socket for monitoring new containers, a few issues arise:
- Mounting container runtime socket inside a container has some security concerns. Because having access to this socket basically grants control to all containers on a node.
- The nature of the polling model ensures that there must be some downtime where new containers are not monitored (polling period). Ideally, we want this time to be 0. However, decreasing the polling period too much might cause performance issues.
- The polling model also includes some unnecessary calls where no new containers are being created which is just wasting CPU time.

To overcome these issues, OCI Hooks can be utilized to transform the way KubeArmor registers new containers from polling model to event driven model where a hook will run with new containers to notify KubeArmor about the new container.

This model addresses the issues of polling model as follows:
- Mounting container runtime UNIX socket is no longer required since KubeArmor now receives new containers from the hook running on host instead of polling the runtime socket.
- New containers now are being monitored with no downtime since hooks run after containers are created but before they start. So, it is always guaranteed that containers will be registered with KubeArmor before they start running. So, 0 downtime.
- KubeArmor will only be notified when new containers are created instead of polling periodically. So, CPU usage should go down.

## Challenges

So, when it comes to using OCI hooks in different Kubernetes environments, a few challenges arise:
- What information needs to be passed to KubeArmor?
- How will we pass this information to KubeArmor?
- How will we inject the hook configuration with new containers created?
- How can we register containers created before KubeArmor?

### Container Data

So, it turned out, in order for KubeArmor to register new containers and monitor them, it needs to know which [namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) the container is inside and also which [AppArmor](https://en.wikipedia.org/wiki/AppArmor) profile it is using. Since the hook will be running on host it can get namespace details from [procfs](https://en.wikipedia.org/wiki/Procfs) using the PID of the container. As for the AppArmor profile it can access that from container [Bundle](https://github.com/opencontainers/runtime-spec/blob/main/bundle.md).

### Communication with KubeArmor

After the hook manages to get container data needed, it needs to send these data to KubeArmor. This can easily be done using UNIX sockets. The idea is KubeArmor can listen on a socket that is mounted on host so the hook can connect to it and pass container data.
The communication is kept very simple. Hooks send containers and the operation to do on these containers (add or remove) and KubeArmor responds with whether it managed to apply this operation or not. KubeArmor response is important because it is used also to synchronize with a hook sending in multiple messages so KubeArmor can parse them with no issues. It also signals whether KubeArmor is ready to receive messages about new containers or is it still waiting for previous containers to be sent first (more on that later).

### Hook Injection

Applying an OCI hook to containers is runtime dependent. In case of CRI-O, it is really simple. CRI-O daemon just watches a specific directory for new hooks and whenever a new one is created, it is applied immediately to new containers.
So, in order to create the hook configuration in this directory, a Kubernetes Job was used to place the configuration and the executable in the right place and from here runtime will take care of applying the configuration and running the hook.

In case of Containerd, it's a bit different since it doesn't support loading hooks from directory similar to CRI-O. There are generally two approaches for that. First, we can change Containerd daemon configuration to use a base container spec with the needed hooks and the daemon will merge this spec with the new containers. However, changing Containerd configuration will require access to host init process socket to restart it which is a bigger security concern than our original issue. The second approach depends on [NRI](https://github.com/containerd/nri) to manage the hook injection. It requires a little bit of setup from the administrator deploying KubeArmor but it's a bit more secure than the first approach. It expects from user to setup NRI and run a plugin that injects hooks into new containers. Something like [this](https://github.com/containerd/nri/tree/main/plugins/hook-injector). Since it requires setup from the administrator running KubeArmor, they will need to opt in to use the hooks model.

### Monitoring Containers Launched Before KubeArmor

The way hooks work is that they are applied on containers created after configuring container runtime. So, containers created before KubeArmor will not be monitored which is obviously a bad thing. So, we came up with a way to ensure that whenever KubeArmor starts it gets all containers currently on the cluster before it starts accepting new containers. This also important to make sure that KubeArmor doesn't receive a message to remove a container that it is not being monitored and to make sure no race happens between a container being deleted and the same container being added which might cause some issues.

The way we did this was when the hook is running after container is created, it checks if the running container is KubeArmor container. If that's the case it forks a process that waits on KubeArmor initializing and listening on a socket and then it starts sending all containers to KubeArmor with a flag indicating that these are containers created before KubeArmor. KubeArmor then sets a flag indicating it got these containers and starts accepting new containers that way ensuring containers are passed in the right order.

## PRs

- Modifying KubeArmor to be event driven and configuring cluster using CRI-O: https://github.com/kubearmor/KubeArmor/pull/1714
- Configuring cluster using Containerd: https://github.com/kubearmor/KubeArmor/pull/1763
- Updating NRI's Hook Injector-plugin to watch directories: https://github.com/containerd/nri/pull/84
- Fix data race in Hooks package used in CRI-O: https://github.com/containers/common/pull/2009

## Future Work

- Develop a more suitable plugin for our use case. It should handle restarts or crashes to make sure no containers are missed during the time where the plugin is not running.
- Benchmark KubeArmor performance to get the difference between the polling model and the event driven model and see the effect of changing models in CPU usage, container monitoring downtime and container start delays.

## Acknowledgments

I want to give special thanks to my mentors Barun, Rudraksh and Akshay for the time and effort they put to guide me through this project. It was really a great learning experience and I believe we achieved great results through their mentorship. So, thank you!

## Contact Me

- [Linkedin](https://www.linkedin.com/in/abdulrahmanelawady/)
- abdoelawady125@gmail.com
