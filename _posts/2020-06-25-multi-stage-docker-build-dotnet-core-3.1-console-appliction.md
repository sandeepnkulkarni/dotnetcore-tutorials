---
layout: post
title: Use multi-stage docker build for creating docker image of trimmed self-contained .Net Core 3.1 console application
date: 2020-06-25 16:00:00 +0530
tags: [.net, core, console, self-contained, trimmed, multi-stage, docker, alpine]
---

## Install .Net Core 3.1 SDK

.Net Core 3.0 added many more features along with option to publish trimmed self-contained applications. However, 3.0 version is already declared EOL (End of Life). Hence we will use 3.1 which is designated as LTS (Long Term Support) release.

Depending on the version of Ubuntu that you have, choose corresponding commands to run.

Run below commands to install .Net Core SDK 3.1 on Ubuntu 16.04:

```
{
  wget https://packages.microsoft.com/config/ubuntu/16.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo apt-get update
  sudo apt-get install -y apt-transport-https
  sudo apt-get update
  sudo apt-get install -y dotnet-sdk-3.1
}
```

Run below commands to install .Net core SDK 3.1 on Ubuntu 18.04:

```
{
  wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo apt-get update
  sudo apt-get install -y apt-transport-https
  sudo apt-get update
  sudo apt-get install -y dotnet-sdk-3.1
}
```

## Create new console application 

Run below command to create new console application with name `hellodotnet31`:

```
$ dotnet new console -o hellodotnet31

# Output

The template "Console Application" was created successfully.

Processing post-creation actions...
Running 'dotnet restore' on hellodotnet31/hellodotnet31.csproj...
  Determining projects to restore...
  Restored /home/hellodotnet31/hellodotnet31.csproj (in 327 ms).

Restore succeeded.
```

A new directory `hellodotnet31` will be created with default Console Application template.

Edit `hellodotnet31/Program.cs` and update message from `Hello World!` to `Hello .Net Core 3.1!`.

Difference between .Net Core 2.1 and 3.1 appliction is the target framework that is defined. You are allowed to publish trimmed self-contained applications with target framework 3.0 onward only. 

## Publishing trimmed self-contained application

.Net Core 3.0 adds two new options for creating a single file trimmed binary of the project. It further reduces the size of the binary significantly. 

So we are adding `/p:PublishTrimmed=true` and `/p:PublishSingleFile=true` to our command for publishing the application like below:

```
$ dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64 /p:PublishTrimmed=true /p:PublishSingleFile=true

# Output

Microsoft (R) Build Engine version 16.6.0+5ff7b0c9e for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  Restored /home/hellodotnet31/hellodotnet31.csproj (in 422 ms).
  hellodotnet31 -> /home/hellodotnet31/bin/Release/netcoreapp3.1/linux-musl-x64/hellodotnet31.dll
  Optimizing assemblies for size, which may change the behavior of the app. Be sure to test after publishing. See: https://aka.ms/dotnet-illink
  hellodotnet31 -> /home/hellodotnet31/bin/Release/netcoreapp3.1/linux-musl-x64/publish/
```

If we take a look at the output under `bin/Release/netcoreapp3.1/linux-musl-x64/publish/`, we will find a single executable `hellodotnet31` and its PDB file. The size of this binary is very small (around 35 MB) compared to without these two options (around 76 MB). So size has reduced by over 50%!!!

We are adding `/p:PublishTrimmed=true` and `/p:PublishSingleFile=true` to our multi-stage Dockerfile as well.

Along with it .Net Core 3.1 further reduced runtime dependencies as well. Hence, we need to use less no. of packages in Alpine than 2.1. Please compare <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/2.1/alpine3.12/amd64/Dockerfile> and <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/3.1/alpine3.12/amd64/Dockerfile>

Final Dockerfile contents:

```
# Create application build

FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env

WORKDIR /source

COPY *.csproj ./

RUN dotnet restore

COPY . ./

RUN dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64 /p:PublishTrimmed=true /p:PublishSingleFile=true

# Create application image

FROM amd64/alpine:3.12

RUN apk add --no-cache \
    ca-certificates \
    # .NET Core dependencies
    krb5-libs libgcc libintl libssl1.1 zlib libstdc++

# Enable detection of running in a container
ENV DOTNET_RUNNING_IN_CONTAINER=true

# Set the invariant mode since icu_libs isn't included (see https://github.com/dotnet/announcements/issues/20)
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true

WORKDIR /app

COPY --from=build-env /source/bin/Release/netcoreapp3.1/linux-musl-x64/publish/ .

ENTRYPOINT ["./hellodotnet31"]
```

Create a .dockerignore with content like below:

```
[Bb]in/
[Oo]bj/
Dockerfile
*.pdb
```

We can run docker build command like below:

```
docker build -t hellodotnet31 .
```

Sample Output:

```
Sending build context to Docker daemon    167MB
Step 1/13 : FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
 ---> 006ded9ddf29
Step 2/13 : WORKDIR /source
 ---> Using cache
 ---> 48b752b4b167
Step 3/13 : COPY *.csproj ./
 ---> 5b9171db8d71
Step 4/13 : RUN dotnet restore
 ---> Running in 641e02115d5d
  Determining projects to restore...
  Restored /source/hellodotnet31.csproj (in 322 ms).
Removing intermediate container 641e02115d5d
 ---> 041b5ed42fc4
Step 5/13 : COPY . ./
 ---> 2d27db689248
Step 6/13 : RUN dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64 /p:PublishTrimmed=true /p:PublishSingleFile=true
 ---> Running in 9d0c64867b3d
Microsoft (R) Build Engine version 16.6.0+5ff7b0c9e for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  Restored /source/hellodotnet31.csproj (in 13.66 sec).
  hellodotnet31 -> /source/bin/Release/netcoreapp3.1/linux-musl-x64/hellodotnet31.dll
  Optimizing assemblies for size, which may change the behavior of the app. Be sure to test after publishing. See: https://aka.ms/dotnet-illink
  hellodotnet31 -> /source/bin/Release/netcoreapp3.1/linux-musl-x64/publish/
Removing intermediate container 9d0c64867b3d
 ---> 9428e5669d17
Step 7/13 : FROM amd64/alpine:3.12
 ---> a24bb4013296
Step 8/13 : RUN apk add --no-cache     ca-certificates     krb5-libs libgcc libintl libssl1.1 zlib     libstdc++
 ---> Using cache
 ---> 71bde714c50d
Step 9/13 : ENV DOTNET_RUNNING_IN_CONTAINER=true
 ---> Using cache
 ---> cbe1a5d7a5cd
Step 10/13 : ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true
 ---> Using cache
 ---> 974aaaa299b8
Step 11/13 : WORKDIR /app
 ---> Using cache
 ---> 8795eae99175
Step 12/13 : COPY --from=build-env /source/bin/Release/netcoreapp3.1/linux-musl-x64/publish/ .
 ---> 501b65cab1dd
Step 13/13 : ENTRYPOINT ["./hellodotnet31"]
 ---> Running in 8cd1b6dc3e4e
Removing intermediate container 8cd1b6dc3e4e
 ---> 65bcaa6a7313
Successfully built 65bcaa6a7313
Successfully tagged hellodotnet31:latest
```

We can verify that the `hellodotnet31` application runs sucessfully using below command:

```
$ docker run --name hellodotnet31 hellodotnet31

# Output

Hello .Net Core 3.1!
```

## Size comparison (3.1 vs 2.1)

```
$ docker images

# Output 

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hellodotnet31       latest              8a4e2ecbf5f8        3 minutes ago       46.4MB
```

When we compare it with earlier self-trimmed image we had created with .Net Core 2.1 using alpine:3.12 as base image, we see that new image is just 46.4 MB in size vs 86.4 MB earlier.
