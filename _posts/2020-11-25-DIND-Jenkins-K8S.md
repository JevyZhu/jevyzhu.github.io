---
layout: post
title: "Build Docker Image In Jenkins Running On Kubernetes"
---

> Just the best way to build docker image in Jenkins pipeline on K8S



# Background

Running Jenkins job in Pod of K8S is a very popular  solution for CI. In my production environment,  it has K8S cluster and  [Jeninks Kubernetes Plugin](https://github.com/jenkinsci/kubernetes-plugin) connecting to it.  

By using pod template provided by that plug-in, whenever a jenkins pipeline job starts, a temporary slave pod being created for it, then destroyed automatically after job finishes. 

This works pretty well. First of all, It saves resources by replying on K8S to dynamically allocate them. Moreover, to get clean and isolated environments to work, a jenkins job just need to configure containers on demand. 

# Problem

But things become a bit more complicated when people need to build docker image as well. Originally one solution was adopted is using fixed nodes, in which docker is installed, as jenkins slave. This works but there are downsides:

* Nodes must have docker installed, which is heavy for our environment.
* No dynamic resource allocation.
* It breaks our pipeline consistency.

So running docker in container, i.e. docker-in-docker is preferred. After investigation, following solutions were raised.



# Use docker:dind

Docker provides an official docker-in-docker image in docker hub.  So we can use it directly in podTemplate from jenkins pipeline. 

**Note**: To use docker-in-docker, the container need to be run as privileged.



## Use Docker In docker:dind

Here is the mini code need to run docker-in-docker.

```groovy
podTemplate(
yaml: """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
"""
) { node(POD_LABEL) {
    container('dind') {
        sh """
            docker pull busybox
        """
    }
}}
```

Please note `--privileged` is a MUST or it will not work.

```
securityContext:
  privileged: true
```

By default, the DOCKER_HOST is not set in above `dind` container. So when command `docker...` was run, it automatically connect to docker daemon by */var/run/docker.sock*.

Here it is running docker command in `dind` container, which has the docker-daemon running in same container also. 

However what if people want to run docker command from other containers in the same pod of `dind`?



## Use Docker From Other Containers

### Through TCP Port

By default, docker::dind has `dockerd-entrypoint.sh` as entry-point. 

The real command that script finally invoked is like following:

```bash
dockerd --host=unix:///var/run/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert /certs/server/ca.pem --tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem

```

So after starting up, it will be listening on `tcp://0.0.0.0:2376`, with tls verification enabled by default.  Since all containers in pod can communicate like in same host, other containers can easily access docker-daemon by `tcp://localhost:2376`.

#### Disable TLS

TLS is boring - to connect to it  the client must provide certificate. Here let's keep it simple at the beginning by removing the tls verification. 

There are two ways to do it: unset environment variable or overwrite entry-point of `dind`.

#### By Environment
`DOCKER_TLS_CERTDIR` is being used by docker:dind to store tls cert, if set to  empty, then tsl would be disabled automatically. I

In following code, the environment is set to empty and also another new container named `demo`, which uses alpine's **docker-cli** client image, is added into pod.

```groovy
podTemplate(
yaml: """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value:
  - name: demo
    image: alpinelinux/docker-cli
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:2375
    command: ["cat"]
    tty: true
"""
) { node(POD_LABEL) {
    container('demo') {
        sh """
            # use docker-daemon in dind container
            # in demo there is a docker client only
            docker pull busybox
        """
    }
}}
```

**Note**: Here the tcp listening port is changed to **`2375`** automatically when `dind` gets up !! So in `demo` container DOCKER_HOST must be set to *tcp://localhost:2375*.



#### By Custom Entry-Point

Remember that default entry-point of docker:dind  is `dockerd-entrypoint.sh`, this script finally call dockerd as a service.

So  instead of setting environment, replacing entry-point with dockerd plus customized parameters is easy and more portable.

```groovy
podTemplate(
yaml: """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
    command: ["/bin/sh","-c"]
    args: ["dockerd --host=tcp://0.0.0.0:8989"]
  - name: demo
    image: alpinelinux/docker-cli
    env:
    - name: DOCKER_HOST
      value: tcp://localhost:8989
    command: ["cat"]
    tty: true
"""
) { node(POD_LABEL) {
    container('demo') {
        sh """
        	# use docker-daemon in dind container
        	# in demo there is a docker client only
            docker pull busybox
        """
    }
}}
```

Here port `8989` is set as the docker-daemon's listen port by passing `--host` value to entry-point which has been replaced by dockerd command in code. Following it `DOCKER_HOST` in `demo` should be set to tcp://localhost:8989.



### Through Unix-Domain Socket

All containers in one pod  are in same host so they can share files in the host. By mounting host's directory and put local socket file there, it is easy to have docker daemon accessible to all containers. Basically here two steps needed:

* Mount a directory in host for all containers in host machine
* Set host parameter pointing to socket file in above directory

```groovy
def shared_mount_point="/dind-only"
def docker_host_sock = "unix://${shared_mount_point}/docker.sock"

podTemplate(cloud: 'kubernetes',
yaml: """
apiVersion: v1
kind: Pod
spec:
  volumes:
  - name: tmp
    emptyDir: {}
    
  containers:
  - name: dind
    image: docker:dind
    env:
    - name: DOCKER_HOST
      value: ${docker_host_sock}
    securityContext:
      privileged: true
    args: ["--host=${docker_host_sock}"]
    volumeMounts:
    - name: tmp
      mountPath: ${shared_mount_point}
      
  - name: demo
    image: alpinelinux/docker-cli
    env:
    - name: DOCKER_HOST
      value: ${docker_host_sock}
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: tmp
      mountPath: ${shared_mount_point}
"""
) { node(POD_LABEL) {
    container('demo') {
        sh """
            docker pull busybox
        """
    }
}}
```

Here the code makes `dind` container to listen on unix-domain socket through file `/dind-only/docker.sock` that shared with `demo` container through mounting temporary directory on `/dind-only` .

## TCP vs Unix-Domain Socket

There is no big difference. But local socket should have better performance because it is more like read/write local file.


# Wait Docker-Daemon Up !

So far it looks pretty good. But actually there is a potential problem: 

What if the `dind` is already up but **docker-daemon not ready** when the pipeline running`docker ...` command? 

K8S provides start-up probe for us to solve it !

For example, to make sure docker daemon is listening on tcp port when pipeline runs into docker, just add followings into yaml of`dind` container:

```yml
- name: dind
    image: docker:dind
  ...
  ...
  startupProbe:
    tcpSocket:
      port: tcp://localhost:8989
    failureThreshold: 10
    initialDelaySeconds: 5
    periodSeconds: 6
```

For unix-domain socket, just use stat command to check if socket file exists.

```yaml
def docker_host_sock = "unix://${shared_mount_point}/docker.sock"
...
...
- name: dind
    image: docker:dind
  ...
  ...
  startupProbe:
    exec:
      command:
      - stat
      - ${docker_host_sock_file}
```



# Security !

Running `dind` in privileged mode is **NOT SAFE**. 

By replacing docker:dind by `docker:dind-rootless`, the `dind` container can run as a normal user - this makes it safer.

**Note**:`docker:dind-rootless` need `--experimental` option. So we MUST add it when using.



# Summary

Docker-In-Docker is easy to run in Jenkins on K8S. The only concern is `--privileged` may bring security issue. Please be careful. Though in experiment phase, the docker:dind-**rootless** is strongly recommended at this moment for production environment.
