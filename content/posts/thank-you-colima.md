+++
title = "Thank you Colima"
date = "2024-03-11"
tags = ["kubernetes"]
+++

# Local Kubernetes with Colima and Nerdctl

This post is a departure from my travel or expat life writing. Today, I'll share my recent experience with setting up a local Kubernetes cluster. 

## The Challenge

On Friday I decided to set up a local k8s cluster on my Mac M2 for local image builds. I don't have a license to use Docker, so I needed to find other ways. 

I started with Multipass and microk8s, which has a registry add-on. With lots of trial and error I never figured out how to push images and then reference them inside of the VM. I pushed to localhost, which works, and referenced the nodeIP in the Kubernetes VM, like this, but got imagePull errors every time.

```yaml
spec:
  containers:
  - name: proxy
    image: 192.168.1.6:5000/proxy:v1.0
    imagePullPolicy: IfNotPresent
```

I tried again using Minikube, which also has a registry add-on, but the same types of errors occurred. As the saying goes. It's always a network problem.

## The Solution: Colima and Nerdctl

Today I found success with [Colima](https://github.com/abiosoft/colima) and Nerdctl. No Docker is needed, and there is no registry add-on required for Kubernetes cluster. It's all just handled via the file system.

I put this stuff in a [bash script](https://github.com/mreider/k8s/blob/main/setup.sh) on my Github.

Here's the idea:

```bash
colima start --kubernetes --arch aarch64 --runtime containerd
```

Build some images and import them into the cluster:

```bash
nerdctl build -t backend:v1.0 ./apps/backend
nerdctl build -t frontend:v1.0 ./apps/frontend
nerdctl build -t proxy:v1.0 ./apps/proxy
nerdctl -n default save -o backend-v1.0.tar backend:v1.0
nerdctl -n default save -o frontend-v1.0.tar frontend:v1.0
nerdctl -n default save -o proxy-v1.0.tar proxy:v1.0
nerdctl -n k8s.io load -i backend-v1.0.tar
nerdctl -n k8s.io load -i frontend-v1.0.tar
nerdctl -n k8s.io load -i proxy-v1.0.tar
```

Then the deployment file:

```yaml
spec:
  containers:
  - name: proxy
    image: proxy:v1.0
    imagePullPolicy: IfNotPresent
```

It works perfectly!