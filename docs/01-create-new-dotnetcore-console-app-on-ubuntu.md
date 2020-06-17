# Create new .Net core console application on Ubuntu

.NET Core SDK allows you to develop apps with .NET Core.

## Install .Net Core SDK

Depending on the version of Ubuntu that you have, choose corresponding commands to run.

Run below commands to install .Net Core SDK 2.1 on Ubuntu 16.04:

```
{
  wget https://packages.microsoft.com/config/ubuntu/16.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo apt-get update
  sudo apt-get install -y apt-transport-https
  sudo apt-get update
  sudo apt-get install -y dotnet-sdk-2.1
}
```

Run below commands to install .Net core SDK 2.1 on Ubuntu 18.04:

```
{
  wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  sudo dpkg -i packages-microsoft-prod.deb
  sudo apt-get update
  sudo apt-get install -y apt-transport-https
  sudo apt-get update
  sudo apt-get install -y dotnet-sdk-2.1
}
```

## Create new console application 

Run below command to create new console application with name `hellodotnet`:

```
dotnet new console -o hellodotnet
```
Sample output:
```
The template "Console Application" was created successfully.

Processing post-creation actions...
Running 'dotnet restore' on hellodotnet/hellodotnet.csproj...
  Restore completed in 438.13 ms for /home/testuser/hellodotnet/hellodotnet.csproj.

Restore succeeded.

```

A new directory `hellodotnet` will be created with default Console Application template.

Edit `hellodotnet/Program.cs` and update message from `Hello World!` to `Hello .Net Core!`.

## Run the application

To run the application with default settings i.e. `Debug` configuration, run below command inside `hellodotnet` directory:

```
dotnet run
```

Sample output:
```
Hello .Net Core!
```

## Build the application with specific configuration

To build the project in Release configuration, run below command inside `hellodotnet` directory:
```
dotnet build --configuration Release
```

Above command does not run the appliction, it only build a project and all of its dependencies.

Sample output:

```
Microsoft (R) Build Engine version 16.2.37902+b5aaefc9f for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 96.51 ms for /home/testuser/hellodotnet/hellodotnet.csproj.
  hellodotnet -> /home/testuser/hellodotnet/bin/Release/netcoreapp2.1/hellodotnet.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:02.26

```

To build the project in Debug configuration, run below command inside `hellodotnet` directory:
```
dotnet build --configuration Debug
```

## Publish the application

Before you want to deploy this application on a hosting system, you need to publish it. 

The output includes the following assets:

* Intermediate Language (IL) code in an assembly with a dll extension.
* A .deps.json file that includes all of the dependencies of the project.
* A .runtimeconfig.json file that specifies the shared runtime that the application expects, as well as other configuration options for the runtime (for example, garbage collection type).
* The application's dependencies, which are copied from the NuGet cache into the output folder.

To do the same, `dotnet publish` requires to be run like below:

```
dotnet publish --configuration Release 
```

Sample output:
```
Microsoft (R) Build Engine version 16.2.37902+b5aaefc9f for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 382.94 ms for /home/testuser/hellodotnet/hellodotnet.csproj.
  hellodotnet -> /home/testuser/hellodotnet/bin/Release/netcoreapp2.1/hellodotnet.dll
  hellodotnet -> /home/testuser/hellodotnet/bin/Release/netcoreapp2.1/publish/

```

Following files will be available in the publish dirctory:

```
hellodotnet.deps.json  hellodotnet.dll  hellodotnet.pdb  hellodotnet.runtimeconfig.json
```

Hosting system need to have .Net Core 2.1 runtime installed to run the application with below command:

```
dotnet hellodotnet.dll
```

## Publish self-contained application

It is possible to publish application as self-contained. Doing so also publishes the .NET Core runtime with your application so the runtime doesn't need to be installed on the target machine.

To create a self-contained along with runtime identifier for Linux x64 (Most desktop distributions like CentOS, Debian, Fedora, Ubuntu, and derivatives), run below command:

```
dotnet publish --configuration Release --self-contained true --runtime linux-x64
```

Sample output:

```
Microsoft (R) Build Engine version 16.2.37902+b5aaefc9f for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 16.5 sec for /home/testuser/hellodotnet/hellodotnet.csproj.
  hellodotnet -> /home/testuser/hellodotnet/bin/Release/netcoreapp2.1/linux-x64/hellodotnet.dll
  hellodotnet -> /home/testuser/hellodotnet/bin/Release/netcoreapp2.1/linux-x64/publish/
```

You will need to distribute all the files under publish directory to the hosting system.

A platform specific executable will also be created in the publish directory which you can use to run your application directory like below:

```
./hellodotnet

## Output
Hello .Net Core!
```

Please check Microsoft webesite under References section below to get list of runtime identifier for other supported OSes.

### References:

* <https://docs.microsoft.com/en-us/dotnet/core/install/linux-ubuntu>
* <https://docs.microsoft.com/en-us/dotnet/core/rid-catalog>
