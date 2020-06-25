---
layout: post
title: Use multi-stage docker build for creating docker image of trimmed self-contained .Net Core 3.1 console application
date: 2020-06-23 14:00:00 +0530
tags: [.net, core, console, self-contained, trimmed, multi-stage, docker, alpine]
---

.Net Core 3.0 adds two new options for creating a single file trimmed binary of the project. It further reduces the size of the binary significantly. 

So we are adding `/p:PublishTrimmed=true` and `/p:PublishSingleFile=true` to our Dockerfile so that we a single `./hellodocker` as output.

Along with it, .Net Core 3.1 further reduced runtime dependencies as well. Hence, we need to use less no. of packages in Alpine than 2.1. Please compare <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/2.1/alpine3.12/amd64/Dockerfile> and <https://github.com/dotnet/dotnet-docker/blob/master/src/runtime-deps/3.1/alpine3.12/amd64/Dockerfile>

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

ENTRYPOINT ["./hellodocker"]
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
docker build -t hellodocker:3.2 .
```

Sample Output:

```
Sending build context to Docker daemon  5.632kB
Step 1/13 : FROM mcr.microsoft.com/dotnet/core/sdk:3.1 AS build-env
 ---> 006ded9ddf29
Step 2/13 : WORKDIR /source
 ---> Using cache
 ---> 48b752b4b167
Step 3/13 : COPY *.csproj ./
 ---> Using cache
 ---> cb72eaf148d7
Step 4/13 : RUN dotnet restore
 ---> Using cache
 ---> 8915785c77ed
Step 5/13 : COPY . ./
 ---> 745bc375221b
Step 6/13 : RUN dotnet publish --configuration Release --self-contained true --runtime linux-musl-x64 /p:PublishTrimmed=true /p:PublishSingleFile=true
 ---> Running in a8b86deb68f9
Microsoft (R) Build Engine version 16.6.0+5ff7b0c9e for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  Restored /source/hellodocker.csproj (in 17.22 sec).
  hellodocker -> /source/bin/Release/netcoreapp3.1/linux-musl-x64/hellodocker.dll
  Optimizing assemblies for size, which may change the behavior of the app. Be sure to test after publishing. See: https://aka.ms/dotnet-illink
  hellodocker -> /source/bin/Release/netcoreapp3.1/linux-musl-x64/publish/
Removing intermediate container a8b86deb68f9
 ---> f5932f3c80c2
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
 ---> ca0539590d77
Step 13/13 : ENTRYPOINT ["./hellodocker"]
 ---> Running in 3ad64c8c11a1
Removing intermediate container 3ad64c8c11a1
 ---> 0ed4eb4ea645
Successfully built 0ed4eb4ea645
Successfully tagged hellodocker:3.2
```

We can verify that the `hellodocker` application runs sucessfully using below command:

```
$ docker run --name hellodocker32 hellodocker:3.2

# Output

Hello Docker!
```

## Size comparison

```
$ docker images hellodocker

# Output 
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
hellodocker         3.2                 0ed4eb4ea645        About a minute ago   46.4MB
```

When we compare it with earlier self-trimmed image we had created with .Net Core 2.1 using alpine:3.12 as base image, we see that new image is just 46.4 MB in size vs 86.4 MB earlier.
