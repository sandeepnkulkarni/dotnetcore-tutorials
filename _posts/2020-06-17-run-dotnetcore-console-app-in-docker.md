---
layout: post
title: Run .Net Core console application inside Docker container
date: 2020-06-17 16:00:00 +0530
---

Refer to [Create your first .Net core console application on Ubuntu](2020-06-17-create-first-dotnetcore-console-app-on-ubuntu.md) to understand in detail how to create your first .Net Core console application on Ubuntu.

I will assume that you already have basic understanding about Docker and have access to a machine with Docker client installed.

## Create and publish .Net Core console application

Create a new console application `hellodocker` with below command:

```
dotnet new console -o hellodocker
```

Edit hellodocker/Program.cs and update message from `Hello World!` to `Hello Docker!`.

Verify that you are able to either run or build the application with `dotnet run` or `dotnet build`.

Now proceed with publishing the application, so that we can use the output to create Docker image.

```
dotnet publish --configuration Release 
```

As we have not mentioned any specific output directory, by default the output will be available inside `/bin/Release/netcoreapp2.1/publish/`

## Create Docker file

We will use mcr.microsoft.com/dotnet/core/runtime:2.1 as base image. This is an official Docker image with .NET Core 2.1 runtime from Microsoft.

We need to create a Docker file where we will start with a base image and then copy the published output from .Net Core console application and the provide entrypoint for the container.

Create a file named `Dockerfile` in directory `hellodocker` and put following content using your favourite editor.

```
FROM mcr.microsoft.com/dotnet/core/runtime:2.1

WORKDIR /app

COPY /bin/Release/netcoreapp2.1/publish/ .

ENTRYPOINT ["dotnet", "hellodocker.dll"]
```

Here we are copying everything inside `/bin/Release/netcoreapp2.1/publish/` into Docker image and specifying the entrypoint where it will run command `dotnet hellodocker.dll`.

## Create Docker image

Docker image can be created with below command:

```
docker build -t hellodocker:1.0 .
```

In above command, with parameter `-t` we have specified that we want to name docker image `hellodocker` and tag as `1.0`.

Sample output:

```
Sending build context to Docker daemon 183.8 kB
Step 1/4 : FROM mcr.microsoft.com/dotnet/core/runtime:2.1
 ---> 6bf090c289b7
Step 2/4 : WORKDIR /app
 ---> Using cache
 ---> ef1c367ba7a9
Step 3/4 : COPY /bin/Release/netcoreapp2.1/publish/ .
 ---> fb8a16c17123
Removing intermediate container 9c904d1f6e06
Step 4/4 : ENTRYPOINT dotnet hellodocker.dll
 ---> Running in 176ba4914624
 ---> be9d0437b610
Removing intermediate container 176ba4914624
Successfully built be9d0437b610
```

You can confirm that the docker image was created with below command:

```
docker images hellodocker:1.0

# Output

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hellodocker         1.0                 be9d0437b610        54 seconds ago      180 MB
```


## Run docker container

To run the docker image as a container, use below command:

```
docker run hellodocker:1.0 --name hellodocker

# Output

Hello Docker!
```

Docker container will exit immediately, because we had specified a very simple console application as entry point which prints a message on console.

## References
* <https://hub.docker.com/_/microsoft-dotnet-core-runtime/>
