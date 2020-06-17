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

References:
<https://docs.microsoft.com/en-us/dotnet/core/install/linux-ubuntu>
