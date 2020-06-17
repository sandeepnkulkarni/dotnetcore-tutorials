# Run .Net Core console application inside Docker container

Refer to [Create new .Net core console application on Ubuntu](01-create-new-dotnetcore-console-app-on-ubuntu.md) to understand in detail how to create .Net Core console application on Ubuntu.

## Create .Net Core console application

Create a new console application `hellodocker` with below command:

```
dotnet new console -o hellodocker
```

Edit hellodocker/Program.cs and update message from `Hello World!` to `Hello Docker!`.

Verify that you are able to either run or build the application with `dotnet run` or `dotnet build`.

