---
layout: post
title: Create self-contained .Net Core console application docker image
date: 2020-06-18 16:00:00 +0530
tags: [.net, core, console, self-contained, docker, ubuntu]
---

Refer to section **Create new console application** in [Create your first .Net Core console application on Ubuntu](./2020-06-17-create-first-dotnetcore-console-app-on-ubuntu.md) to understand basics about how to create .Net Core console application. We will re-use the application created here.

## Enabling the Globalization Invariant Mode

Open hellodocker.csproj and enable Globalization Invariant Mode by adding following line under `<PropertyGroup>` tag.

```
<InvariantGlobalization>true</InvariantGlobalization>
```

Resultant hellodocker.csproj:

```
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp2.1</TargetFramework>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>

</Project>
```

If we don't enable Globalization Invariant Mode setting, we will fail with below error when we run it inside a container:

```
FailFast:
Couldn't find a valid ICU package installed on the system. Set the configuration flag System.Globalization.Invariant to true if you want to run with no globalization support.

   at System.Environment.FailFast(System.String)
   at System.Globalization.GlobalizationMode.GetGlobalizationInvariantMode()
   at System.Globalization.GlobalizationMode..cctor()
   at System.Globalization.CultureData.CreateCultureWithInvariantData()
   at System.Globalization.CultureData.get_Invariant()
   at System.Globalization.CultureInfo..cctor()
   at System.StringComparer..cctor()
   at System.AppDomain.InitializeCompatibilityFlags()
   at System.AppDomain.Setup(System.Object)
```

There are two other approaches to enable globalization invariant mode:

1. In runtimeconfig.json file by adding `"System.Globalization.Invariant": true` like below:

```
{
    "runtimeOptions": {
        "configProperties": {
            "System.Globalization.Invariant": true
        }
    }
}
```

2. Setting environment variable `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT` to `true` or `1` inside the docker file like below:

```
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT true
```

Please refer to **.NET Core Globalization Invariant Mode** document mentioned under [References](#references) below.

In case you like to use Globalization feature instead, you will need to install the ICU package inside the docker image.

## Publish self-contained application

To create a self-contained along with runtime identifier for Linux x64 (suitable for most desktop distributions like CentOS, Debian, Fedora, Ubuntu, and derivatives), run below command:

```
dotnet publish --configuration Release --self-contained true --runtime linux-x64
```

Sample output:

```
Microsoft (R) Build Engine version 16.2.37902+b5aaefc9f for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 441.66 ms for /home/testuser/hellodocker/hellodocker.csproj.
  hellodocker -> /home/testuser/hellodocker/bin/Release/netcoreapp2.1/linux-x64/hellodocker.dll
  hellodocker -> /home/testuser/hellodocker/bin/Release/netcoreapp2.1/linux-x64/publish/
```

It will have around 184 files (72 MB) in the output directory `bin/Release/netcoreapp2.1/linux-x64/publish/` because it contains the .NET Core runtime along with your application.

## Create Docker image

In other article [Run .Net Core console application inside Docker container](2020-06-17-run-dotnetcore-console-app-in-docker.md), we have used mcr.microsoft.com/dotnet/core/runtime:2.1 as base image. It was done because we were in need of .Net Core runtime which allows us to run application.

But this time, we can use any Linux docker image as base because we have published self-contained application. We will use ubuntu:18.04 and compile the Docker image.

```
FROM ubuntu:18.04

WORKDIR /app

COPY /bin/Release/netcoreapp2.1/linux-x64/publish/ .

ENTRYPOINT ["./hellodocker"]
```

Docker image can be created with below command:

```
docker build -t hellodocker:2.0 .
```

In above command, with parameter -t we have specified that we want to name docker image hellodocker and tag as 2.0.

Sample output:
```
Sending build context to Docker daemon 76.11 MB
Step 1/4 : FROM ubuntu:18.04
 ---> 8e4ce0a6ce69
Step 2/4 : WORKDIR /app
 ---> Using cache
 ---> b26d48ea93c9
Step 3/4 : COPY /bin/Release/netcoreapp2.1/linux-x64/publish/ .
 ---> d454fc295dda
Removing intermediate container 3c7cc8f26d7b
Step 4/4 : ENTRYPOINT ./hellodocker
 ---> Running in d05ba4f6e320
 ---> 9c47434bb6f7
Removing intermediate container d05ba4f6e320
Successfully built 9c47434bb6f7
```


## Run docker container

To run the docker image as a container, use below command:

```
docker run --name hellodocker2 hellodocker:2.0

# Output
Hello Docker!
```


## Docker image size difference

As we have used a different base image and included only the required .Net Core runtime for the application, docker image size is smaller. This can be verified by running `docker images` command:

```
$ docker images

# Output

REPOSITORY         TAG                 IMAGE ID            CREATED             SIZE  
hellodocker        2.0                 9ff4eeed6631        2 minutes ago       138 MB
hellodocker        1.0                 0a19597e8d5e        24 hours ago        180 MB
```

If we use smaller docker images like Alpine (or for that matter any slim docker image), size will reduce even further. We will look into it in next blog post.

## Disabling the Globalization Invariant Mode

Now that we know how to enable Globalization Invariant Mode and create self-contained application, we will also try to install ICU packages during creation of Docker image.

We will remove below line added to hellodocker.csproj:

```
<InvariantGlobalization>true</InvariantGlobalization>
```

and publish a new build again with:

```
dotnet publish --configuration Release --self-contained true --runtime linux-x64
```

We can refer to <https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md> and get the ICU package name that we need to use within our Docker image. We can see that we need to install `libicu60` and `openssl1.0`. So our new Dockerfile will be:

```
FROM ubuntu:18.04

RUN apt-get update && apt-get install -y libicu60 openssl1.0

WORKDIR /app

COPY /bin/Release/netcoreapp2.1/linux-x64/publish/ .

ENTRYPOINT ["./hellodocker"]
```

After that we can create our new docker image with:

```
docker build -t hellodocker:2.1 .
```

We are adding a tag `2.1` to differntiate it from earlier images we created.

```
$ docker run --name hellodocker21 hellodocker:2.1

# Output

Hello Docker!
```

We can do a size comparison again now:

```
$ docker images

# Output

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hellodocker         2.1                 48e8d067af2f        4 minutes ago       244MB
hellodocker         2.0                 9ff4eeed6631        4 days ago          138MB
hellodocker         1.0                 0a19597e8d5e        5 days ago          180MB
```

As we have added new packages to run .Net Core application with Globalization support, size of docker image has increased significantly.

## References

* <https://hub.docker.com/_/ubuntu>
* <https://docs.microsoft.com/en-us/dotnet/core/run-time-config/globalization>
* <https://github.com/dotnet/runtime/blob/master/docs/design/features/globalization-invariant-mode.md>
* <https://github.com/dotnet/core/blob/master/Documentation/linux-prereqs.md>
