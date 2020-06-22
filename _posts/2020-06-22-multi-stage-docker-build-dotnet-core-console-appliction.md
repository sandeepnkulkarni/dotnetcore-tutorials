---
layout: post
title: Use multi-stage docker build for creating docker image of self-contained .Net Core console application
date: 2020-06-22 14:00:00 +0530
tags: [.net, core, console, self-contained, multi-stage, docker, alpine]
---

In this tutorial, we will use multi-stage Docker builds to illustrate how we can build an application in the first stage and in next stage, use the output from first stage to create an application image. A multi-stage build is done by creating different sections of a Dockerfile, each referencing a different base image. This allows a multi-stage build to fulfill a function previously filled by using multiple docker files, copying files between containers, or running different pipelines.

Refer to section **Create new console application** in [Create your first .Net Core console application on Ubuntu](2020-06-17-create-first-dotnetcore-console-app-on-ubuntu.md#create-new-console-application) to understand basics about how to create .Net Core console application. We will re-use the application created here.

## Multi-stage docker file

In the application build stage, we will use `mcr.microsoft.com/dotnet/core/sdk:2.1` as base image. We will copy source code inside the container and publish a self-contained application. We will use `linux-musl-x64` as runtime configuration since we are using Apline as base image as mentioned below. Output for this stage will be available in `/source/bin/Release/netcoreapp2.1/linux-musl-x64/publish/`.

Next stage we will create application image which will consume the output from build stage. We will use `amd64/alpine:3.12` as base image this time. It is a very small image with size 5.57 MB. However, to be able to sucessfully run .Net Core applications, we will need to install few packages. The list of packages is mentioned in <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/2.1/alpine3.12/amd64/Dockerfile>

We will also add two environment variables `DOTNET_RUNNING_IN_CONTAINER` and `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT` and set them to `true` as suggested by Microsoft. 

`DOTNET_RUNNING_IN_CONTAINER` variable is used to detect whether the .Net Core application is running inside the docker.

`DOTNET_SYSTEM_GLOBALIZATION_INVARIANT` is used to enable Globalization Invariant Mode. In earlier tutorial we had used by adding following line under <PropertyGroup> tag in hellodocker.csproj:

```
<InvariantGlobalization>true</InvariantGlobalization>
```

Hence, we can remove that line from hellodocker.csproj this time.

Final Dockerfile contents:

```
# Create application build

FROM mcr.microsoft.com/dotnet/core/sdk:2.1 AS build-env

WORKDIR /source

COPY *.csproj ./

RUN dotnet restore

COPY . ./

RUN dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64

# Create application image

FROM amd64/alpine:3.12

RUN apk add --no-cache \
    ca-certificates \
    # .NET Core dependencies
    krb5-libs libgcc libintl libssl1.1 zlib \
    libstdc++ lttng-ust tzdata userspace-rcu

# Enable detection of running in a container
ENV DOTNET_RUNNING_IN_CONTAINER=true

# Set the invariant mode since icu_libs isn't included (see https://github.com/dotnet/announcements/issues/20)
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true

WORKDIR /app

COPY --from=build-env /source/bin/Release/netcoreapp2.1/linux-musl-x64/publish/ .

ENTRYPOINT ["./hellodocker"]
```

To increase the build's performance, exclude files and directories, create a `.dockerignore` file in the same directory as `Dockerfile` with following contents:

```
Dockerfile
[b|B]in/
[O|o]bj/
```

Now that our `Dockerfile` is ready along with `.dockerignore` file, we can run `docker build` command like below:

```
docker build -t hellodocker:3.0 .
```

We have tagged the image with `3.0`, so that we can differntiate it with earlier docker images created earlier for hellodocker application. We will use the earlier images to do a size comparison later.

Sample output from `docker build` command:

```
Sending build context to Docker daemon  5.632kB
Step 1/13 : FROM mcr.microsoft.com/dotnet/core/sdk:2.1 AS build-env
 ---> 156e5cc5d7a3
Step 2/13 : WORKDIR /source
 ---> Using cache
 ---> c3afbc8b54b3
Step 3/13 : COPY *.csproj ./
 ---> Using cache
 ---> 7066245f5486
Step 4/13 : RUN dotnet restore
 ---> Using cache
 ---> 1bd61a20491a
Step 5/13 : COPY . ./
 ---> Using cache
 ---> f7be8f770396
Step 6/13 : RUN dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64
 ---> Using cache
 ---> ed14de370bee
Step 7/13 : FROM amd64/alpine:3.12
 ---> a24bb4013296
Step 8/13 : RUN apk add --no-cache     ca-certificates     krb5-libs libgcc libintl libssl1.1 zlib     libstdc++ lttng-ust tzdata userspace-rcu
 ---> Running in 34c216b91c78
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.12/community/x86_64/APKINDEX.tar.gz
(1/12) Installing ca-certificates (20191127-r4)
(2/12) Installing krb5-conf (1.0-r2)
(3/12) Installing libcom_err (1.45.6-r0)
(4/12) Installing keyutils-libs (1.6.1-r1)
(5/12) Installing libverto (0.3.1-r1)
(6/12) Installing krb5-libs (1.18.2-r0)
(7/12) Installing libgcc (9.3.0-r2)
(8/12) Installing libintl (0.20.2-r0)
(9/12) Installing libstdc++ (9.3.0-r2)
(10/12) Installing userspace-rcu (0.12.1-r0)
(11/12) Installing lttng-ust (2.12.0-r1)
(12/12) Installing tzdata (2020a-r0)
Executing busybox-1.31.1-r16.trigger
Executing ca-certificates-20191127-r4.trigger
OK: 14 MiB in 26 packages
Removing intermediate container 34c216b91c78
 ---> 343db9e44ba9
Step 9/13 : ENV DOTNET_RUNNING_IN_CONTAINER=true
 ---> Running in 76d1e455c34f
Removing intermediate container 76d1e455c34f
 ---> 98de9a9abf35
Step 10/13 : ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true
 ---> Running in 93927d2a8f4d
Removing intermediate container 93927d2a8f4d
 ---> d0682cb9935a
Step 11/13 : WORKDIR /app
 ---> Running in 85d1cb292ed3
Removing intermediate container 85d1cb292ed3
 ---> 012ed99836ff
Step 12/13 : COPY --from=build-env /source/bin/Release/netcoreapp2.1/linux-musl-x64/publish/ .
 ---> 0f55de06ca79
Step 13/13 : ENTRYPOINT ["./hellodocker"]
 ---> Running in 0d16235fb56d
Removing intermediate container 0d16235fb56d
 ---> bb6dd22b46f3
Successfully built bb6dd22b46f3
Successfully tagged hellodocker:3.0
```

We can verify that the `hellodocker` application still runs without any issue with below command:

```
$ docker run --name hellodocker3 hellodocker:3.0

#Output

Hello Docker!
```

## Docker size comparison

Now let us try to do a size comparison for docker images created earlier with the latest image:

```
$ docker images

# Output

REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
hellodocker         3.0                 67bdd4ac2207        About a minute ago   86.4MB
hellodocker         2.0                 9ff4eeed6631        3 days ago           138MB
hellodocker         1.0                 0a19597e8d5e        4 days ago           180MB
```

`hellodocker:1.0` was created with `mcr.microsoft.com/dotnet/core/runtime:2.1` as base image. This base image itself has size of around 180 MB. As our application was not published as self-contained, it required .Net runtime installed to run. Appllication size was only 24 KB.

`hellodocker:2.0` was created with `ubuntu:18.04` as base image. This base image still has size of around 64 MB. Our application was published as self-contained, it did not require .Net runtime installed to run. However, application size had grown to 71.2 MB. Hence, the resultant size of image was 138 MB.

`hellodocker:3.0` we have created just now uses `alpine:3.12` as base image. This base image is just 5.57 MB in size. But in order to run .Net Core applications, we have to install few packages as mentioned in the `Dockerfile`. Here the size of docker image is just 86.4 MB. That is a significant reduction in size in comparison to 180 MB.

```
$ docker history hellodocker:3.0

# Output

IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
67bdd4ac2207        3 minutes ago       /bin/sh -c #(nop)  ENTRYPOINT ["./hellodocke…   0B
2bcdb10bc1b0        3 minutes ago       /bin/sh -c #(nop) COPY dir:ab880fd822eea1843…   74.2MB
437dce78118a        3 minutes ago       /bin/sh -c #(nop) WORKDIR /app                  0B
4338bc2a6eec        3 minutes ago       /bin/sh -c #(nop)  ENV DOTNET_SYSTEM_GLOBALI…   0B
0cd34d5a50d8        3 minutes ago       /bin/sh -c #(nop)  ENV DOTNET_RUNNING_IN_CON…   0B
a55da04ad332        3 minutes ago       /bin/sh -c apk add --no-cache     ca-certifi…   6.66MB
a24bb4013296        3 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B
<missing>           3 weeks ago         /bin/sh -c #(nop) ADD file:c92c248239f8c7b9b…   5.57MB
```

In .Net Core 3.0 onward, there is an additional option `p:PublishTrimmed` to further reduce the size of self-contained executable. We will cover that option as well in another tutorial. You can refer to link given in [References](#references) section below to take a look at it.

## References

* <https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md>
* <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/2.1/alpine3.12/amd64/Dockerfile>
