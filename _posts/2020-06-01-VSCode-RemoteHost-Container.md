---
layout: post
title: "Setup VSCode use Container on Remote Host
---

> A quick start to use  **container on remote host  ** for development.

# Prepare Remote Host
## Install Docker Engine
Refer to docker official [document](https://docs.docker.com/engine/install/)
## Accept external connection
On the **remote host** running docker.
1. Create `daemon.json` file in `/etc/docker`:
   ```bash
    {"hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]}
   ```
2. Add `/etc/systemd/system/docker.service.d/override.conf`
   ```bash
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd
   ```
3. Reload the systemd daemon:

   ```bash
    systemctl daemon-reload
   ```
4. Restart docker:
   ```bash
    systemctl restart docker.service
   ```

# Prepare Desktop
## Install Open SSH Client
### For Linux/Mac
Use package manager to install
### For Windows
1. Download OpenSSH binary for windows from [here](https://github.com/PowerShell/Win32-OpenSSH/releases)

2. Unzip it to a directory, e.g. C:\OpenSSH

3. Add C:\OpenSSH to Path environment 

4. Open **cmd** as administrator, run command
   ```bash
   C:> ssh-agent 
   ```
   to verify it can be found.

## Start ssh-agent
### For Linux/Mac
Most of time it has ssh-agent automatically running if not run command:
```bash
 eval "$(ssh-agent -s)"
```
### For Windows
In `PowerShell` as administrator:

```bash
Set-Service ssh-agent -StartupType Automatic
```
Then open [Windows Task Manger] --> [Services] Tab -> Make the ssh-agent started

**Note:** for Win10, it can use [build-in openssh](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse) to replace above steps.

## Generate ssh key

Run `ssh-keygen` to generate key pair.

```bash
ssh-keygen
```
By default following prompt it will generate public/private keys in `%USERPROFILE%/.ssh` and named as:
* `id_rsa`
* `id_rsa.pub`

Upload `id_rsa.pub` to docker host  as  `~/.ssh/authorized_keys`, and verify key pair work fine:
```bash
ssh -i id_rsa <user>@<host>
```
Now client should be able to login the remote host without password prompt.

## Setup Docker 

### Install docker
#### For Linux/Mac

Use package manager to install

#### For Windows
Choose one of options:
* Install [docker desktop for windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows), -- but it **not necessary!** 
* Just download [docker.exe](https://github.com/StefanScherer/docker-cli-builder/releases/) and put it into a directory of %Path% environment.


###  Set up docker context (optional)
Open `cmd` as administrator and create docker context
```bash
docker context create <context name> --docker "host=ssh://<user>@<host>"
```
then switch docker context to it:
```bash
docker context use <context name>
```
verify it works by run:
```
docker info
```
it should output same as running `docker info` on remote host.


## Visual Studio Code
### Install VSCode

Download  [VSCode](https://code.visualstudio.com/) and install

Open VSCode:
* Extensions -> Search Remote -> Install `Remote Development`
* Extensions -> Search Remote -> Install `Docker`

### Switch Docker context in VSCode
Press `ctrl+shift+p` to issue command `docker contexts use` , from prompt list choose the development context created from above.

If the docker client has already setup context to use remote host docker, this step can be ignored.

#  Start Project !

> There are **two ways** to use remote host container

## Option 1 - Attach Remote Host Container

### Make project in Remote Host

Login remote docker host and then create a empty directory like `~/newprj`, then add a Dockerfile in it. Here it is a python project.

```dockerfile
FROM python                                                                
ENV DEBIAN_FRONTEND=noninteractive 
ENV HTTP_PROXY "http://web-proxy.ap.softwaregrp.net:8080"                                           
ENV HTTPS_PROXY "http://web-proxy.ap.softwaregrp.net:8080" 
RUN mkdir -p /etc/apt && touch /etc/apt/apt.conf \                                                  
    && echo "Acquire::http::proxy \"http://web-proxy.ap.softwaregrp.net:8080/\";" > /etc/apt/apt.conf \                                                                                                 
    && echo "Acquire::https::proxy \"http://web-proxy.ap.softwaregrp.net:8080/\";" >> /etc/apt/apt.conf \                                                                                               
    && apt-get update 
# Switch back to dialog for any ad-hoc use of apt-get                                               
ENV DEBIAN_FRONTEND=                 
```

Run docker command to build the image and start container as daemon.
```bash
docker build . -t <image tag>
docker run --name <container name> -it  -v ~/newprj:/workspaces/newprj <image tag>
```
**Note:** Attach volume is very necessary to make sure any change in container can  be saved to host's disk.
**Note: do NOT ** use "-d" to make container running as daemon or VSCode will not attach to it.

### Attach to the container
Open VSCode and `ctrl+shift+p` to issue  command`Remote-Containers:Attach to Running Container...`, select the name of container just created above. 
Now can open folder/files in container and start python development as usual even no python installed locally or in remote host! Everything is in docker!

There is no any project-related source file locally in this way.

## Option 2 - Start from Desktop

### Add Dockerfile and Config in Local
1. Create a local folder on Desktop and add a file named Dockerfile in it with content as above dockerfile.
2. Add file `devcontainer/devcontainer.json` in ther folder, with config like following:
```json
{
	"name": "Demo",
	"context": "..",
	"dockerFile": "../Dockerfile",
	"workspaceFolder": "/workspace",
	"workspaceMount": "source=<RemoteHostDir>,target=/workspace,type=bind,consistency=cached",
}
```
Here the <RemoteHostDir> is the absolute directory of remote host  to mount when container starts, e.g. `/home/demo/newproj`. Please note it is impossble to mount local file system.
### Make project in Remote Host
```bash
mkdir /home/demo/newproj
```
### Open in Remote Container
Open VSCode, `File`->`Open Folder...`, choose the local folder contains above `devcontainer/devcontainer.json`.

Because devcontainer.json file configed, VSCode will prompt asking if open in remote container, just click "yes" it will **automatically** build, start and attach container on remote host!

In this way, it has Dockerfile and VS config locally but other source code remotely. Files may have to be sync from time to time in development.



# Extra-links

1.  VSCode [Advanced Container Configuration](https://code.visualstudio.com/docs/remote/containers-advanced)
2. GitHub [microsoft/vscode-dev-containers](https://github.com/microsoft/vscode-dev-containers)
3. GitHub [microsoft/vscode-remote-try-python](https://github.com/microsoft/vscode-remote-try-python)

