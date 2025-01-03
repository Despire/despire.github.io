---
layout: post
title: "Containers, What are they ?"
date: 2025-01-02
categories: [Tech]
---


When I first learned about containers a few years ago, I was introduced to think of them as a kind of lightweight virtual machine with a shared kernel. At that time I didn't really think much about it because I knew how virtual machines worked at a very high level, so the term "lightweight virtual machine with a shared kernel" stuck with me for many years and I didn't really think much about it because I was mostly working with the high level API of containers, such as running, downloading, attaching to a container, and so on. 

It was a long time before I had to dive into the internals of containers and how they work to solve problems I had. I won't bother you with the description of the problems as they are mostly boring and not worth talking about, but what I think is worth it is the internal of how these containers work.

At a very high level, traditional virtual machines work by actually running a complete operating system (known as the guest OS) that includes a different kernel, which is different from the operating system that runs on the actual hardware (known as the host OS). You can install many different operating systems on your computer, each of which is completely independent of the host OS, with its own set of binaries and applications. You can limit the computing resources on these deployed "virtual machines" and control exactly how much each can consume.

Containers, on the other hand, share the same underlying kernel, and the only things that are "isolated" are the binaries and applications that are deployed. Essentially, what containers give us is the ability to isolate our applications from each other while also controlling the resource consumption that each application is allowed to consume.

These two concepts are illustrated in the figure below, where a traditional virtual machine is shown on the left and containers are shown on the right.

![alt text](/assets/img/containers/vm.png)

Okay, great, now that we have the high-level descriptions of what containers are, what are they really? I mean, what do they use to achieve this per-application isolation? Well, let's dive into a Docker container runtime and try to reverse engineer to see what's being done to achieve that.

There is a great video by Liz Race from [GOTO 2018](https://www.youtube.com/watch?v=8fi7uSYlOdc) where she goes through the very basics of what containers are. Mostly I'm going to scratch the same surface, but I'll also try to add more things to get a bigger picture.

We'll be using [Docker](https://www.docker.com/) as the container runtime. I highly recommend that you run the examples in the following sections on a completely new virtual machine running Linux, as we will be running commands with privileges and you don't want to accidentally destroy your environment.

Lets start by downloading an image with `docker pull` 
**NOTE**: all of the examples are done on a fresh VM just docker installed.

```bash
$ docker pull registry.k8s.io/e2e-test-images/agnhost:2.39
2.39: Pulling from e2e-test-images/agnhost
0eeab5c20069: Pull complete 
8be7f7d96b31: Pull complete 
b119ab3a30e8: Pull complete 
9d50e66b1b79: Pull complete 
737a5fb8286b: Pull complete 
d5ade2ed934a: Pull complete 
efe82aafeb7c: Pull complete 
e6d4e4bda6fd: Pull complete 
df3266e97453: Pull complete 
2e5f0eb2f34c: Pull complete 
Digest: sha256:7e8bdd271312fd25fc5ff5a8f04727be84044eb3d7d8d03611972a6752e2e11e
Status: Downloaded newer image for registry.k8s.io/e2e-test-images/agnhost:2.39
registry.k8s.io/e2e-test-images/agnhost:2.39

```

Wait, what is an image anyway? Well lets have a look. Docker stores anything relevant under the `/var/lib/docker`

```bash
$ cd /var/lib/docker/
$ ls
buildkit  containers  engine-id  image  network  overlay2  plugins  runtimes  swarm  tmp  volumes

```

There are a lot of directories, where do we look ? well from the above above it seems that we downloaded a total of 10 "of something". Docker also printed a hash next to the downloaded items so lets search for the last one downloaded inside the `/var/lib/docker` directory.

```bash
$ grep -r "2e5f0eb2f34c" ./

./image/overlay2/distribution/v2metadata-by-diffid/sha256/4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a:[{"Digest":"sha256:2e5f0eb2f34cf44850e0c439266ddb5f0240a526d53adb2614985c69679b935b","SourceRepository":"registry.k8s.io/e2e-test-images/agnhost","HMAC":""}]

```

We found a JSON file that contains the hash, and whats interesting is the path of the file 
```
./image/overlay2/distribution/v2metadata-by-diffid/sha256/4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a
```

It seems that the path contains a different hash that references our originally searched for hash.
So lets look for any files containing this new hash.

```bash
$ grep -r "4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a" ./

./image/overlay2/layerdb/sha256/1c7a1bab7d8e974b098055c75576325d0850b1752a390f5ebdfa198902a55a00/diff:sha256:4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a

./image/overlay2/distribution/diffid-by-digest/sha256/2e5f0eb2f34cf44850e0c439266ddb5f0240a526d53adb2614985c69679b935b:sha256:4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a

./image/overlay2/imagedb/content/sha256/972b6f1c939916b5576d3cba6efca45f421c46603bc9a25dbf02309c5bba820d:{"architecture":"arm64","config":{"ExposedPorts":{"5000/tcp":{},"80/tcp":{},"8080/tcp":{},"8081/tcp":{},"9376/tcp":{}},"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Entrypoint":["/agnhost"],"Cmd":["pause"],"Labels":{"commit_id":"1290d6a4c0c5cc4f758b428993d2c3760d9cf279","git_url":"https://github.com/kubernetes/kubernetes/tree/1290d6a4c0c5cc4f758b428993d2c3760d9cf279/test/images/agnhost","image_version":"2.39"},"ArgsEscaped":true,"OnBuild":null},"created":"2022-05-25T13:45:11.628020281Z","history":[{"created":"2022-04-04T23:39:53.434744204Z","created_by":"/bin/sh -c #(nop) ADD file:a2a992b7f6af1e6f8f5648f329f4a4058d8c4377417ac23ae211290c0cdc8f4b in / "},{"created":"2022-04-04T23:39:53.555422536Z","created_by":"/bin/sh -c #(nop)  CMD [\"/bin/sh\"]","empty_layer":true},{"created":"2022-05-25T13:45:04.002578655Z","created_by":"COPY qemu-aarch64-static /usr/bin/ # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:08.582515982Z","created_by":"RUN /bin/sh -c apk --update add bind-tools curl netcat-openbsd iproute2 iperf bash \u0026\u0026 rm -rf /var/cache/apk/*   \u0026\u0026 ln -s /usr/bin/iperf /usr/local/bin/iperf   \u0026\u0026 ls -altrh /usr/local/bin/iperf # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:08.657166174Z","created_by":"ADD https://github.com/coredns/coredns/releases/download/v1.6.2/coredns_1.6.2_linux_arm64.tgz /coredns.tgz # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.100558648Z","created_by":"RUN /bin/sh -c tar -xzvf /coredns.tgz \u0026\u0026 rm -f /coredns.tgz # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.245730882Z","created_by":"EXPOSE map[5000/tcp:{} 80/tcp:{} 8080/tcp:{} 8081/tcp:{} 9376/tcp:{}]","comment":"buildkit.dockerfile.v0","empty_layer":true},{"created":"2022-05-25T13:45:11.245730882Z","created_by":"RUN /bin/sh -c mkdir /uploads # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.280737116Z","created_by":"ADD porter/localhost.crt localhost.crt # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.324707484Z","created_by":"ADD porter/localhost.key localhost.key # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.478955727Z","created_by":"ADD agnhost agnhost # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.628020281Z","created_by":"RUN /bin/sh -c ln -s agnhost agnhost-2 # buildkit","comment":"buildkit.dockerfile.v0"},{"created":"2022-05-25T13:45:11.628020281Z","created_by":"ENTRYPOINT [\"/agnhost\"]","comment":"buildkit.dockerfile.v0","empty_layer":true},{"created":"2022-05-25T13:45:11.628020281Z","created_by":"CMD [\"pause\"]","comment":"buildkit.dockerfile.v0","empty_layer":true}],"moby.buildkit.buildinfo.v1":"eyJmcm9udGVuZCI6ImRvY2tlcmZpbGUudjAiLCJzb3VyY2VzIjpbeyJ0eXBlIjoiZG9ja2VyLWltYWdlIiwicmVmIjoiZG9ja2VyLmlvL2FybTY0djgvYWxwaW5lOjMuMTIiLCJwaW4iOiJzaGEyNTY6YWZjODQ1ZmQ0Y2ViNWE5MDQwODcwNjY1NjdkZjBjMjIzMjkyOTI5MGU2NDJjNDdjOTIwMjI3ZTY1ZGM5OTM3YyJ9LHsidHlwZSI6Imh0dHAiLCJyZWYiOiJodHRwczovL2dpdGh1Yi5jb20vY29yZWRucy9jb3JlZG5zL3JlbGVhc2VzL2Rvd25sb2FkL3YxLjYuMi9jb3JlZG5zXzEuNi4yX2xpbnV4X2FybTY0LnRneiIsInBpbiI6InNoYTI1NjpmZWIyNTQyMzYxNjgzOTg4YWIzNDBhY2RiODFiYWE5N2EwZDEwODA5ODJlNmNlNGRlNTBkZjRmZGNmNjg1OGE0In1dfQ==","os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:d80e0208345a5c0e0e5575f11f35d99a179bcdfec9a075828e774145c0245eb6","sha256:d43ff6376af873ae711f7b32a6a0040b3e000de6fc6fdf4e0d9b0b81d895be48","sha256:dd2b5085c8d8dc4229980b836b97e5237f346edf51f374a85965598af8bac3e8","sha256:6ab0139fa8b5b1676f441ec4e6b059c75e2cb44f05857e64976063fd2e9a2096","sha256:f5bbfc3846d8c612f62791b8029e28ebe2fbdc205e13ab3d559e2a48134dc9d1","sha256:264449c0d0c579bacddad637cebf6c73add5645015c9944029a04eb1cd5ce56f","sha256:c13090a6183ac17dd665af4d226e8a1929a9d24336737794985eb91b758f8bce","sha256:d81907e7b9ff929379a9d243416fdb44510cd718ed2a633b9b93ef0595cf3dfe","sha256:90ee4f69b5790e9b289312a2bdd112d47d05b47f05e8f0bea210f5d4cd17e171","sha256:4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a"]}}
```

We have found 3 references and the last one seems to be a JSON so lets pretty print it.

```
{
  "architecture": "arm64",
  "config": {
    "ExposedPorts": {
      "5000/tcp": {},
      "80/tcp": {},
      "8080/tcp": {},
      "8081/tcp": {},
      "9376/tcp": {}
    },
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Entrypoint": [
      "/agnhost"
    ],
    "Cmd": [
      "pause"
    ],
    "Labels": {
      "commit_id": "1290d6a4c0c5cc4f758b428993d2c3760d9cf279",
      "git_url": "https://github.com/kubernetes/kubernetes/tree/1290d6a4c0c5cc4f758b428993d2c3760d9cf279/test/images/agnhost",
      "image_version": "2.39"
    },
    "ArgsEscaped": true,
    "OnBuild": null
  },
  "created": "2022-05-25T13:45:11.628020281Z",
  "history": [
    {
      "created": "2022-04-04T23:39:53.434744204Z",
      "created_by": "/bin/sh -c #(nop) ADD file:a2a992b7f6af1e6f8f5648f329f4a4058d8c4377417ac23ae211290c0cdc8f4b in / "
    },
    {
      "created": "2022-04-04T23:39:53.555422536Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
      "empty_layer": true
    },
    {
      "created": "2022-05-25T13:45:04.002578655Z",
      "created_by": "COPY qemu-aarch64-static /usr/bin/ # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:08.582515982Z",
      "created_by": "RUN /bin/sh -c apk --update add bind-tools curl netcat-openbsd iproute2 iperf bash && rm -rf /var/cache/apk/*   && ln -s /usr/bin/iperf /usr/local/bin/iperf   && ls -altrh /usr/local/bin/iperf # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:08.657166174Z",
      "created_by": "ADD https://github.com/coredns/coredns/releases/download/v1.6.2/coredns_1.6.2_linux_arm64.tgz /coredns.tgz # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.100558648Z",
      "created_by": "RUN /bin/sh -c tar -xzvf /coredns.tgz && rm -f /coredns.tgz # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.245730882Z",
      "created_by": "EXPOSE map[5000/tcp:{} 80/tcp:{} 8080/tcp:{} 8081/tcp:{} 9376/tcp:{}]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    },
    {
      "created": "2022-05-25T13:45:11.245730882Z",
      "created_by": "RUN /bin/sh -c mkdir /uploads # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.280737116Z",
      "created_by": "ADD porter/localhost.crt localhost.crt # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.324707484Z",
      "created_by": "ADD porter/localhost.key localhost.key # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.478955727Z",
      "created_by": "ADD agnhost agnhost # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.628020281Z",
      "created_by": "RUN /bin/sh -c ln -s agnhost agnhost-2 # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2022-05-25T13:45:11.628020281Z",
      "created_by": "ENTRYPOINT [\"/agnhost\"]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    },
    {
      "created": "2022-05-25T13:45:11.628020281Z",
      "created_by": "CMD [\"pause\"]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    }
  ],
  "moby.buildkit.buildinfo.v1": "eyJmcm9udGVuZCI6ImRvY2tlcmZpbGUudjAiLCJzb3VyY2VzIjpbeyJ0eXBlIjoiZG9ja2VyLWltYWdlIiwicmVmIjoiZG9ja2VyLmlvL2FybTY0djgvYWxwaW5lOjMuMTIiLCJwaW4iOiJzaGEyNTY6YWZjODQ1ZmQ0Y2ViNWE5MDQwODcwNjY1NjdkZjBjMjIzMjkyOTI5MGU2NDJjNDdjOTIwMjI3ZTY1ZGM5OTM3YyJ9LHsidHlwZSI6Imh0dHAiLCJyZWYiOiJodHRwczovL2dpdGh1Yi5jb20vY29yZWRucy9jb3JlZG5zL3JlbGVhc2VzL2Rvd25sb2FkL3YxLjYuMi9jb3JlZG5zXzEuNi4yX2xpbnV4X2FybTY0LnRneiIsInBpbiI6InNoYTI1NjpmZWIyNTQyMzYxNjgzOTg4YWIzNDBhY2RiODFiYWE5N2EwZDEwODA5ODJlNmNlNGRlNTBkZjRmZGNmNjg1OGE0In1dfQ==",
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:d80e0208345a5c0e0e5575f11f35d99a179bcdfec9a075828e774145c0245eb6",
      "sha256:d43ff6376af873ae711f7b32a6a0040b3e000de6fc6fdf4e0d9b0b81d895be48",
      "sha256:dd2b5085c8d8dc4229980b836b97e5237f346edf51f374a85965598af8bac3e8",
      "sha256:6ab0139fa8b5b1676f441ec4e6b059c75e2cb44f05857e64976063fd2e9a2096",
      "sha256:f5bbfc3846d8c612f62791b8029e28ebe2fbdc205e13ab3d559e2a48134dc9d1",
      "sha256:264449c0d0c579bacddad637cebf6c73add5645015c9944029a04eb1cd5ce56f",
      "sha256:c13090a6183ac17dd665af4d226e8a1929a9d24336737794985eb91b758f8bce",
      "sha256:d81907e7b9ff929379a9d243416fdb44510cd718ed2a633b9b93ef0595cf3dfe",
      "sha256:90ee4f69b5790e9b289312a2bdd112d47d05b47f05e8f0bea210f5d4cd17e171",
      "sha256:4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a"
    ]
  }
}

```

Okay, so the found JSON contains some information about the downloaded docker image, and at the bottom of the file is the hash we were looking for, stored under a type called "layers".

```bash
$ docker pull registry.k8s.io/e2e-test-images/agnhost:2.39
2.39: Pulling from e2e-test-images/agnhost
0eeab5c20069: Pull complete 
8be7f7d96b31: Pull complete 
b119ab3a30e8: Pull complete 
9d50e66b1b79: Pull complete 
737a5fb8286b: Pull complete 
d5ade2ed934a: Pull complete 
efe82aafeb7c: Pull complete 
e6d4e4bda6fd: Pull complete 
df3266e97453: Pull complete 
2e5f0eb2f34c: Pull complete 
Digest: sha256:7e8bdd271312fd25fc5ff5a8f04727be84044eb3d7d8d03611972a6752e2e11e
Status: Downloaded newer image for registry.k8s.io/e2e-test-images/agnhost:2.39
registry.k8s.io/e2e-test-images/agnhost:2.39
```
So essentially what this `docker pull` does it seems to download a bunch of layers into the host file system. But what actually are these layers ?

One of the other paths that was found referencing our hash was 
```bash
./image/overlay2/layerdb/sha256/1c7a1bab7d8e974b098055c75576325d0850b1752a390f5ebdfa198902a55a00/diff:sha256:4d698fdb75002056847ee0d1ffcbc8a746d5e5ceebc2b3358235eee0f1348f9a
```

So lets list whats inside that directory

```bash
$ ls ./image/overlay2/layerdb/sha256/1c7a1bab7d8e974b098055c75576325d0850b1752a390f5ebdfa198902a55a00/

cache-id  diff  parent  size  tar-split.json.gz

```
There are a bunch of hashes inside each of these files if you would dump them via `cat` and it you would then grep these hashes there is one that stands out and is stored inside the `cache-id` file


which in my case is
```bash
$ cat ./image/overlay2/layerdb/sha256/1c7a1bab7d8e974b098055c75576325d0850b1752a390f5ebdfa198902a55a00/cache-id 

5bf06ed7a43fa779b3c6d6162fc22c8a88540d210cf24106ee1e7f4125cdf035
```

As I was looking around for the references for this hash I found that there is a directory inside `/var/lib/docker/overlay2` with exactly this hash

```bash
$ ls overlay2/5bf06ed7a43fa779b3c6d6162fc22c8a88540d210cf24106ee1e7f4125cdf035/

committed  diff  link  lower  work
```

Now, depending on how familiar you are with Linux, there are already hints as to what these "layers" actually are. They are a pseudo filesystem built using the `overlay` mount, these hints are also in the file path of the respective files.

Again, I don't know any of the internals of Docker, so everything I'm doing is just trying to reverse engineer the bigger picture of how containers work.

Overlay filesystem is a union filesystem implementation in the Linux kernel that allows one directory tree (the **upper** layer) to be overlaid on another directory tree (the **lower** layer).

- Lower Layer - Is typically read-only, it serves as a base that cannot be modified.
- Upper layer - Is read and write, any changes made to files in the lower layer are stored in the upper layer instead.
- Merged - These two layers are merged into a single union view of these two layers, which is the one we usually interact with.

So what happens when we run these downloaded Docker images?

```bash
$ docker run registry.k8s.io/e2e-test-images/agnhost:2.39
Paused

```

In another terminal we can list the mounts and check for an overlay filesystem

```bash
$ mount 

overlay on /var/lib/docker/overlay2/0dc9e11caffe1f941a358b7ab0aade2cd7f1c532ca49dc65f9c3be729e42a43c/merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/VAJBN2IW7SZ7TRTPGXPNCOGMTG:/var/lib/docker/overlay2/l/6DTDIDT4PHBY7JAC23ZFCNKLWN:/var/lib/docker/overlay2/l/2RCNRU2ZJ5IEMC4M62HM5PLNEI:/var/lib/docker/overlay2/l/Y75UYVFC7V3E7Y4PHRQYSNBRY4:/var/lib/docker/overlay2/l/JM24ZBE7HGIJWI3FKXEUSU5NVO:/var/lib/docker/overlay2/l/GA52BDTG6YTDJNY7KEFDYXXSUZ:/var/lib/docker/overlay2/l/VEQI25EFL2WC2L7QZURGQMQP76:/var/lib/docker/overlay2/l/BD36D7YSKP5BO7KPPEQYXZX7TN:/var/lib/docker/overlay2/l/KHIG4GOP3TWMC5YD7IKYZ5CQTH:/var/lib/docker/overlay2/l/6I7HXYXD56K4UELL2TBLQONO7P:/var/lib/docker/overlay2/l/NG3PYEJWS4DQSX7RS6UXDLVXV4,upperdir=/var/lib/docker/overlay2/0dc9e11caffe1f941a358b7ab0aade2cd7f1c532ca49dc65f9c3be729e42a43c/diff,workdir=/var/lib/docker/overlay2/0dc9e11caffe1f941a358b7ab0aade2cd7f1c532ca49dc65f9c3be729e42a43c/work,nouserxattr)

...

```

From the output you can see where the `merged` view is stored, the `lower` layer and the `upper` layer. If you then execute inside the container and look at the contents of these directories, they should match (note that the `lower' layer seems to be a bunch of links that you would have to follow to get to the actual `lower' layer being used).

Great! Now we know that when we download a Docker image, we are downloading pre-built layers that will be mounted when the image is transformed into a container.

Now if we remove the downloaded image 

```bash
$ docker rm $(docker ps -aq)
$ docker rmi registry.k8s.io/e2e-test-images/agnhost:2.39
Untagged: registry.k8s.io/e2e-test-images/agnhost:2.39
Untagged: registry.k8s.io/e2e-test-images/agnhost@sha256:7e8bdd271312fd25fc5ff5a8f04727be84044eb3d7d8d03611972a6752e2e11e
Deleted: sha256:972b6f1c939916b5576d3cba6efca45f421c46603bc9a25dbf02309c5bba820d
Deleted: sha256:1c7a1bab7d8e974b098055c75576325d0850b1752a390f5ebdfa198902a55a00
Deleted: sha256:df99b80dc1d512caee7840b4514dfece8d68269242d197500ca0000ec48714da
Deleted: sha256:be312e0af147d017fcd24a6865e723b009f3d9c2123a1771b48772164c838e2d
Deleted: sha256:20c6d02912c753d099d2410eceffc3ce2e97a7cb456dfb36d670c98c3b8f7616
Deleted: sha256:8420a6f4b81ebf4cfd35e7d7107fca5bfce0f965d34d0d965fc5b29b1044b736
Deleted: sha256:a91990b1e31361e6221cdb1a2447360eeae8e97190fa173f5dbb15f4837604e7
Deleted: sha256:4795b8e10f5634330ed7f1b37b577adb88b0aadbae2249ecc1aedc5ca1b83650
Deleted: sha256:96ddf3fd7a6fc7e598e9e639aa4278fd1573f68a79258939ed4d40c916f11bcb
Deleted: sha256:dc4e823607d3db78a06c2252e1af57a85151165af5be8401b861db4ab131fac7
Deleted: sha256:d80e0208345a5c0e0e5575f11f35d99a179bcdfec9a075828e774145c0245eb6
```

Notice again that we are deleting the previously downloaded layers and if we look inside the `overlay2` directory under `/var/lib/docker` it should be empty, expect for the `l` which contains some kind of link, but since we don't have any images present on the machine it should be empty as well.

```bash
$ ls /var/lib/docker/overlay2
l

$ ls /var/lib/docker/overlay2/l/

```

Now, lets download a simpler image with only one layer.

```bash
docker pull ubuntu
```

Now if we look inside the `overlay2` directory under `/var/lib/docker`

```bash
$ ls overlay2/13fed57b58b2e355399d33ab2c87b75249c9e35ec85bb333bf295d3191cf55e5/

diff  link

$ ls overlay2/13fed57b58b2e355399d33ab2c87b75249c9e35ec85bb333bf295d3191cf55e5/diff/

bin  boot  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```


We can actually see the ubuntu filesystem that is mounted when the container is started. If you look back at the first figure of this article, the separated `bin/libs` and the `app` part on top of them should now make more sense, essentially when we start a container we mount an overlay filesystem that contains the necessary binaries and libs and restrict the container from seeing anything else. Again, since I don't know the internals of how Docker manages this, there may be a lot more "stuff" going on, but the core part of mounting an overlay file system is there.

Now, the question is how do we restrict what the container can and cannot see ? This is where namespaces come in

Namespaces are part of the Linux kernel for years and allow you to create your own namespace for various resources. If we look at the help manual of the `unshare` command on linux

```bash
$ unshare -h

Usage:
 unshare [options] [<program> [<argument>...]]

Run a program with some namespaces unshared from the parent.

Options:
 -m, --mount[=<file>]      unshare mounts namespace
 -u, --uts[=<file>]        unshare UTS namespace (hostname etc)
 -i, --ipc[=<file>]        unshare System V IPC namespace
 -n, --net[=<file>]        unshare network namespace
 -p, --pid[=<file>]        unshare pid namespace
 -U, --user[=<file>]       unshare user namespace
 -C, --cgroup[=<file>]     unshare cgroup namespace
 -T, --time[=<file>]       unshare time namespace

```

It lists all the supported namespaces that we can create and then move our "container" into those namespaces. This will limit what the container sees and interacts with.

But there is more, how does networking work with docker? Lets run the downloaded ubuntu image and install the `iproute2` package and list the available interfaces inside the container.

```bash
$ docker run -it ubuntu
root@9ccbe8351f81:/# apt update && apt install iproute2 && ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
33: eth0@if34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever



```

It looks like Docker created a `veth` in the namespace of the running container, but since `veths` are created in pairs (think of them as a virtual ethernet cable connecting two endpoints), the other end of the `veth` must be in a different namespace. If we look at the interfaces on the host

```bash
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 6a:eb:5b:1c:62:51 brd ff:ff:ff:ff:ff:ff
    inet 192.168.72.2/24 brd 192.168.72.255 scope global dynamic noprefixroute enp0s1
       valid_lft 2428sec preferred_lft 2428sec
    inet6 fd31:bfa4:d197:8537:6675:b0d:5a33:5e43/64 scope global temporary dynamic 
       valid_lft 589230sec preferred_lft 70790sec
    inet6 fd31:bfa4:d197:8537:346d:9fbd:5c72:17f6/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 2591967sec preferred_lft 604767sec
    inet6 fe80::127f:30a2:be8e:5939/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:45:4d:61:e1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:45ff:fe4d:61e1/64 scope link 
       valid_lft forever preferred_lft forever
34: veth3060149@if33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether d6:da:e6:21:f7:6e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::d4da:e6ff:fe21:f76e/64 scope link 
       valid_lft forever preferred_lft forever

```

We will find the other matching `veth` called `veth3060149@if33`, but not only that, we can also see that it is connected to a `bridge` interface called `docker0`. A bridge interface is used to inter-connect all container interfaces allowing communication among them and the host.Additionally, what is interesting is that the interface of the `veth` pair in the host namespace does not have an IP address assigned. Since it is connected to the bridge interface it does not need it, as the bridge operates on the L2 MAC addresses.

If we look again inside the container at the routes 

```bash
$ ip route
default via 172.17.0.1 dev eth0 
172.17.0.0/16 dev eth0 proto kernel scope link src 172.17.0.2
```

and at the route on the host

```bash
$ ip route
default via 192.168.72.1 dev enp0s1 proto dhcp metric 100 
169.254.0.0/16 dev enp0s1 scope link metric 1000 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.72.0/24 dev enp0s1 proto kernel scope link src 192.168.72.2 metric 100 
```

We can see that any ingress or egress traffic related to this container goes through this created `veth` pair, which is then connected to the `bridge` interface and from there is handled by the OS routing tables to communicate with other services either on the host or on the Internet.

We could envision this topology as shown below.

![alt text](/assets/img/containers/network.png)


As the last thing, because the docker network is really a private network, known only within the host machine, any communication that is going outside it would need to use a NAT and forwarding and indeed if we look at the NAT and forwarding rules on the host there are ones created for docker specifically

```
$ iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N DOCKER
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A DOCKER -i docker0 -j RETURN
```

```
$ iptables -S
-P INPUT ACCEPT
-P FORWARD DROP
-P OUTPUT ACCEPT
-N DOCKER
-N DOCKER-ISOLATION-STAGE-1
-N DOCKER-ISOLATION-STAGE-2
-N DOCKER-USER
-A FORWARD -j DOCKER-USER
-A FORWARD -j DOCKER-ISOLATION-STAGE-1
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
-A DOCKER-ISOLATION-STAGE-1 -j RETURN
-A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
-A DOCKER-ISOLATION-STAGE-2 -j RETURN
-A DOCKER-USER -j RETURN
       
```

Great! Now that we seem to have a picture of what a container looks like from the inside, lets build one from scratch, using the concepts we have learned so far

- Its own file system with its own binaries, libraries.
- Its own namespaces.
- Networking to communicate with other containers and the host.

<br>
<hr>
<br>

Again, the code will use concepts from the talk of [Liz rice](https://www.youtube.com/watch?v=8fi7uSYlOdc) Which at this point you should really watch before continuing.

What we are going to build is an API similar to the `docker run -it ubuntu` which will create a `bash` shell with the mounted overlay filesystem in its own namespace. But we will implement a very limited version of this, as anything larger would start to approach on the territory of more robust containers, which is definitely not the goal of this article. 

The goal is simply to become more familiar with what Docker is under the hood. The code is made available on [github](https://github.com/Despire/playground/tree/ffcfb21a273ad44f1f669fd102cd68be131ea04a/tinycontainer). The container is implemented in roughly 360 lines of Go code which consists of the above mentioned principles used in containers.

For the rest of this article we will first show how to run the container and then briefly explain what each section of code does.

From this point, I'm under the assumption that there are no running docker containers on your system nor any images are downloaded.
To start, lets download the github repository with `git clone git@github.com:Despire/playground.git` and exec into the `tinycontainer` directory inside it and initialize it with `make`.

```bash
$ cd ./playground/tinycontainer
$ make
```

Next download the ubuntu image with `docker pull ubuntu` and move the contents of the downloaded image layer from `/var/lib/docker/overlay2/ddafbc0ad29c2940a9aff6bd56fddb8c1f380e0c7673c64a310cf430ba40bbd0/diff/` into `./overlay/image/`

```bash
$ cp -r /var/lib/docker/overlay2/ddafbc0ad29c2940a9aff6bd56fddb8c1f380e0c7673c64a310cf430ba40bbd0/diff/* ./overlay/image/
```
**NOTE**: The hash may be completely different for you, thus you need to check for yourself.

Build the program and run it

```bash
go build .
./container run bash
```

You should have a working shell that is inside the container. The abilities within it are very limited but you can check that the network is working with

```bash
getent hosts www.google.com

2a00:1450:4014:80e::2004 www.google.com
```

While the network connection is working, you will not be able to run `apt update` or install any packages. This is because the mounted overlay filesystem is not complete, for example there are no devices in `/dev`. But as I said, that was not the goal of this article. The goal was to set up a tiny workable container, and that's what we did, anything else would start to go into the robustness of the container implementation, which is out of scope.

Below is also an asciinema video going through the invididual steps.

<script src="https://asciinema.org/a/da2fhZFKe8OMF7lQ9LiYRT7sO.js" id="asciicast-da2fhZFKe8OMF7lQ9LiYRT7sO" async></script>


```go
func main() {
	if err := run(os.Args[1:]); err != nil {
		log.Fatalf("failed to run container: %v", err)
	}
}

func run(args []string) error {
	if len(args) < 1 {
		return fmt.Errorf("no arguments recieved, expected <run> <args...>")
	}

	fmt.Printf("pid: %v\n", os.Getpid())

	switch args[0] {
	case "run":
		return prepare(args[1:])
	case "interactive":
		return interact(args[1:])
	default:
		return fmt.Errorf("unrecognized command: %v, expected usage <run> <args...>", args[0])
	}
}

func prepare(args []string) error {
	if len(args) < 1 {
		return fmt.Errorf("no container arguments, expected <args...>")
	}

	runtime.LockOSThread()
	defer runtime.UnlockOSThread()

	this, newns, err := setupNetworkNamespaces()
	if err != nil {
		return fmt.Errorf("failed to setup network namespaces: %w", err)
	}
	defer newns.Close()
	defer this.Close()
	defer netns.Set(this)

	cleanup, err := setupContainerNetwork(this, newns)
	defer cleanup()
	if err != nil {
		return fmt.Errorf("failed to setup container networking: %w", err)
	}

	umount, err := createOverlay(lower, upper, work, merged)
	defer umount()
	if err != nil {
		return fmt.Errorf("failed to create overlay fs: %w", err)
	}

	cmd := exec.Command("/proc/self/exe", append([]string{"interactive"}, args...)...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.Stdin = os.Stdin
	cmd.Env = []string{
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		fmt.Sprintf("TERM=%s", os.Getenv("TERM")),
	}
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags:   syscall.CLONE_NEWUTS | syscall.CLONE_NEWNS | syscall.CLONE_NEWPID,
		Unshareflags: syscall.CLONE_NEWNS,
	}
	return cmd.Run()
}

func interact(args []string) error {
	if len(args) < 1 {
		return fmt.Errorf("no container arguments, expected <args...>")
	}

	if err := syscall.Sethostname([]byte("container")); err != nil {
		return fmt.Errorf("failed to change container hostname: %w", err)
	}

	if err := syscall.Chroot(merged); err != nil {
		return fmt.Errorf("failed to chroot inside container: %w", err)
	}

	if err := syscall.Chdir("/"); err != nil {
		return fmt.Errorf("failed to change dir to '/' inside container: %w", err)
	}

	if err := syscall.Mount("proc", "/proc", "proc", 0, ""); err != nil {
		return fmt.Errorf("failed to mount proc dir inside container: %w", err)
	}

	defer func() {
		if err := syscall.Unmount("/proc", 0); err != nil {
			fmt.Printf("failed to unmount proc inside container: %w", err)
		}
	}()

	if err := os.WriteFile("/etc/resolv.conf", []byte(etcResolve), 0644); err != nil {
		return fmt.Errorf("failed to inject resolve.conf: %w", err)
	}

	fmt.Printf("executing command: %v with args: %v\n", args[0], args[1:])

	cmd := exec.Command(args[0], args[1:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	return cmd.Run()
}
```

The above code example has two functions, `prepare` and `interact`. The former sets up the namespaces and the
filesystem for the container and proceeds to run itself under these new namespaces, where it then sets up the container environment by
then sets up the container environment by changing the hostname, setting the chroot, and mounting relevant files/directories.

Creating the overlay filesystem is a simple syscall

```go
func createOverlay(lower, upper, work, merged string) (func() error, error) {
	// mount -t overlay overlay -o lowerdir=./image/,upperdir=./container/upper/,workdir=./container/work/ ./container/merged/
	err := syscall.Mount(
		lower,
		merged,
		"overlay",
		0,
		fmt.Sprintf("lowerdir=%s,upperdir=%s,workdir=%s", lower, upper, work),
	)
	if err != nil {
		return func() error { return nil }, err
	}

	return func() error { return syscall.Unmount(merged, 0) }, nil
}
```

Most of the work is done in setting up the network. To simplify the logic a bit, we used some dependencies
for netlink, iptables, and network namespaces.


```go
var (
	myDockerInterfacePostroutingRules = [...][]string{
		// allow communicating with outside world with NAT.
		// iptables -t nat -A POSTROUTING -s 'my-docker0-ip' ! -o my-docker0 -j MASQUERADE
		[]string{"-s", gatewayAddr, "!", "-o", myDockerInterface, "-j", "MASQUERADE"},
	}

	myDockerInterfaceForwardRules = [...][]string{
		// allow packets that are part of established connections to be forwarded.
		// iptables -A FORWARD -o my-docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
		[]string{"-o", myDockerInterface, "-m", "conntrack", "--ctstate", "RELATED,ESTABLISHED", "-j", "ACCEPT"},
		// allow forwarding packets originating from my-docker0 to be forwarded to other interfaces.
		// iptables -A FORWARD -i my-docker0 ! -o my-docker0 -j ACCEPT
		[]string{"-i", myDockerInterface, "!", "-o", myDockerInterface, "-j", "ACCEPT"},
		// allow communication within containers.
		// iptables -A FORWARD -i my-docker0 -o my-docker0 -j ACCEPT
		[]string{"-i", myDockerInterface, "-o", myDockerInterface, "-j", "ACCEPT"},
	}
)

func setupContainerNetwork(parent, child netns.NsHandle) (func() error, error) {
	var (
		cleanupVeth   = func() error { return nil }
		cleanupBridge = func() error { return nil }
		cleanupFunc   = func() error {
			var all error
			all = errors.Join(all, cleanupVeth())
			all = errors.Join(all, cleanupRules())
			all = errors.Join(all, cleanupBridge())
			return all
		}
	)

	// create bridge
	la := netlink.NewLinkAttrs()
	la.Name = "my-docker0"
	myDockerBridge := &netlink.Bridge{
		LinkAttrs: la,
	}

	if err := netlink.LinkAdd(myDockerBridge); err != nil {
		return cleanupFunc, fmt.Errorf("failed to setup my-docker0 bridge: %w", err)
	}

	cleanupBridge = func() error { return netlink.LinkDel(myDockerBridge) }

	bridgeAddr, err := netlink.ParseAddr(gatewayAddr)
	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to parse my-docker0 bridge cidr: %w", err)
	}

	if err := netlink.AddrAdd(myDockerBridge, bridgeAddr); err != nil {
		return cleanupFunc, fmt.Errorf("failed to set addr for my-docker0 bridge: %w", err)
	}

	if err := netlink.LinkSetUp(myDockerBridge); err != nil {
		return cleanupFunc, fmt.Errorf("failed to bring up my-docker0 bridge: %w", err)
	}

	ipt, err := iptables.New()
	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to create iptables client: %w", err)
	}

	for _, r := range myDockerInterfacePostroutingRules {
		if err := ipt.Append("nat", "POSTROUTING", r...); err != nil {
			return cleanupFunc, fmt.Errorf("failed to create NAT POSTROUTING rule %v: %w", r, err)
		}
	}

	for _, r := range myDockerInterfaceForwardRules {
		if err := ipt.Append("filter", "FORWARD", r...); err != nil {
			return cleanupFunc, fmt.Errorf("failed to create ip FORWARD rule %v: %w", r, err)
		}
	}

	// create veth pair
	vethLa := netlink.NewLinkAttrs()
	vethLa.Name = "veth0"
	veth0 := &netlink.Veth{
		LinkAttrs: vethLa,
		PeerName:  "veth1",
	}

	if err := netlink.LinkAdd(veth0); err != nil {
		return cleanupFunc, fmt.Errorf("failed to setup container veth pair: %w", err)
	}

	cleanupVeth = func() error { return netlink.LinkDel(veth0) }

	veth1, err := netlink.LinkByName("veth1")
	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to find veth01: %w", err)
	}

	if err := netlink.LinkSetUp(veth1); err != nil {
		return cleanupFunc, fmt.Errorf("failed to bring up veth1: %w", err)
	}

	if err := netlink.LinkSetMaster(veth1, myDockerBridge); err != nil {
		return cleanupFunc, fmt.Errorf("failed to set my-docer0 bridge as master for veth1: %w", err)
	}

	// move veth0 into the container network namespace configure
	// it, bring up the loopback interface within that namespace
	// and set default route via created bridge.

	if err := netlink.LinkSetNsFd(veth0, int(child)); err != nil {
		return cleanupFunc, fmt.Errorf("failed to move veth0 to container network namespace: %w", err)
	}

	if err := netns.Set(child); err != nil {
		return cleanupFunc, fmt.Errorf("failed to switch to container network namespace: %w", err)
	}

	cleanupVeth = func() error {
		var all error
		all = errors.Join(all, netlink.LinkDel(veth0))
		all = errors.Join(all, netns.Set(parent))
		return all
	}

	veth0Addr, err := netlink.ParseAddr(veth0Addr)
	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to parse veth0 addr: %w", err)
	}

	if err := netlink.AddrAdd(veth0, veth0Addr); err != nil {
		return cleanupFunc, fmt.Errorf("failed to assign address %s to veth0: %w", veth0Addr, err)
	}

	if err := netlink.LinkSetUp(veth0); err != nil {
		return cleanupFunc, fmt.Errorf("failed to bring up veth0: %w", err)
	}

	lo, err := netlink.LinkByName("lo")
	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to find loopback interface inside container network namespace: %w", err)
	}

	if err := netlink.LinkSetUp(lo); err != nil {
		return cleanupFunc, fmt.Errorf("failed to bring up loopback interface inside container network namespace: %w", err)
	}

	err = netlink.RouteAdd(&netlink.Route{
		Scope: netlink.SCOPE_UNIVERSE,
		Gw:    net.ParseIP(gatewayAddr[:len(gatewayAddr) - 2]),
	})

	if err != nil {
		return cleanupFunc, fmt.Errorf("failed to add my-docker0 bridge as default route inside container network namespace: %w", err)
	}

	return cleanupFunc, nil
}
```

When setting up the networking environment for the container, we essentially copy the Docker environment we figured out earlier.
We set up a Linux bridge and create a `veth` pair for each container, where one end of the pair is in its own namespace and the other is in the host
of the bridge. But it does not end there, we need to set up the FORWARDING from the Linux bridge and also set up a NAT entry as this container network is only known to the host, so if the packet would be directed outside the host network it would be dropped.

And finally, you may notice that we copy `/etc/resolv.conf` after entering the container namespace. Since we have mounted the new overlay filesystem as the container's chroot, it does not have this file and does not know how to resolve DNS requests, so we inject a very simple `resolv.conf` file that forwards the request to the host OS for resolution.

```go
	etcResolve = `
nameserver 192.168.72.1
search .
`
```

However, if you were to query some domains with the `getent` binary, you would be able to resolve them. For example, if you tried to install new packages or do an `apt update`, it would fail. This is because some of the directories under the overlay filesystem are missing some links to the host OS. For example, the `/dev` directory is completely empty. Making this work would require adding more robustness and useability into the code.
