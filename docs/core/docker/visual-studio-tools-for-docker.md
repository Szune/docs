---
title: Visual Studio Tools for Docker
description: Using Visual Studio Tools for Docker 
keywords: .NET, .NET Core, Docker, ASP.NET Core, Visual Studio 2015
author: spboyer
manager: wpickett
ms.date: 09/16/2016
ms.topic: article
ms.prod: .net-core
ms.technology: .net-core-technologies
ms.devlang: dotnet
ms.assetid: 1f3b9a68-4dea-4b60-8cb3-f46164eedbbf
---

# Visual Studio Tools for Docker 
Developing and debugging your application in a Docker container can be a ceremony of tasks to get setup with various tools. [Visual Studio Tools for Docker](https://visualstudiogallery.msdn.microsoft.com/0f5b2caa-ea00-41c8-b8a2-058c7da0b3e4) helps you get past the hurdles and find the bugs using F5 to debug your application directly in a locally hosted Docker Container. 

> [!NOTE]
>The current version targets Linux Docker containers, with Windows Containers coming soon.

## Prerequisites
- [Microsoft Visual Studio 2015 Update 3](https://www.visualstudio.com/downloads/download-visual-studio-vs)
- [.NET Core 1.0.1 - VS 2015 Tooling Preview 2](https://go.microsoft.com/fwlink/?LinkID=827546)
- [Docker For Windows](https://www.docker.com/products/docker#/windows) to run your Docker containers locally

## Installation and setup
Download and install the [Visual Studio Tools for Docker](https://visualstudiogallery.msdn.microsoft.com/0f5b2caa-ea00-41c8-b8a2-058c7da0b3e4) from the [Visual Studio Gallery](http://visualstudiogallery.msdn.microsoft.com/) or you can search for it in **Extensions and Updates** within Visual Studio. 

A required configuration is to setup **[Shared Drives](https://docs.docker.com/docker-for-windows/#/shared-drives)** in Docker for Windows. The setting is required for the volume mapping and debugging support.

Right click the Docker icon in the System Tray, click Settings and select Shared Drives.

![Shared Drives](./media/visual-studio-tools-for-docker/settings-shared-drives-win.png) 

## Create an ASP.NET Web Application and add Docker Support

Using Visual Studio, create a new ASP.NET Core Web Application. When the application is loaded, either select **Add Docker Support** from the **Project Menu** or right click the project from the Solution Explorer and select **Add** > **Docker Support**.

Project Menu

![Project Add Docker Support](./media/visual-studio-tools-for-docker/project-add-docker-support.png)

Project Context Menu

![Right Click Add Docker Support](./media/visual-studio-tools-for-docker/right-click-add-docker-support.png)

The following files are added to the project.

- **Dockerfile**: the Docker file for ASP.NET Core applications is based on the [microsoft/aspnetcore](https://hub.docker.com/r/microsoft/aspnetcore) image. This image includes the ASP.NET Core NuGet packages, which have been pre-jitted improving startup performance. When building .NET Core Console Applications, the Dockerfile FROM will reference the most recent [microsoft/dotnet](https://hub.docker.com/r/microsoft/dotnet) image.
- **docker-compose.yml**: base Docker Compose file
- **docker-compose.dev.debug.yml**: Docker Compose file with additional settings for ASPNETCORE_ENVIRONMENT, volume mapping and NuGet packages
- **docker-compose.dev.release.yml**: Docker Compose file for settings in release mode, a subset of those in debug.

The docker-compose.yml file contains the name of the image that is created when project is run. 

```
version '2'

services:
  hellodockertools:
    image:  user/hellodockertools${TAG}
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "80"

``` 

In this example, `image: user/hellodockertools${TAG}` generates the image `user/hellodockertools:dev` when the application is run in **Debug** mode and `user/hellodockertools:latest` in **Release** mode respectively. 

You will want to change the `user` to your Docker Hub username if you plan to push the image to the registry. For example, `spboyer/hellodockertools`, or remove the `user/` for private registries depending on your configuration.

### Debugging
Select **Docker** from the debug dropdown in the toolbar and use F5 to start debugging the application. 

- The microsoft/aspnetcore image is acquired (if not already in your cache)
- ASPNETCORE_ENVIRONMENT is set to Development within the container
- PORT 80 is EXPOSED and mapped to 32769 on your localhost
- Your application is copied to the container
- Default browser is launched with the debugger attached to the container 

The resulting Docker image built is the `dev` image of your application with the `microsoft/aspnetcore` images as the base image.

```console
REPOSITORY                  TAG         IMAGE ID            CREATED         SIZE
spboyer/hellodockertools    dev         0b6e2e44b3df        4 minutes ago   268.9 MB
microsoft/aspnetcore        1.0.1       189ad4312ce7        5 days ago      268.9 MB
```

The application is running using the container which you can see by running the `docker ps` command.

```console
CONTAINER ID        IMAGE                          COMMAND               CREATED             STATUS              PORTS                   NAMES
3f240cf686c9        spboyer/hellodockertools:dev   "tail -f /dev/null"   4 minutes ago       Up 4 minutes        0.0.0.0:32769->80/tcp   hellodockertools_hellodockertools_1
```

### Edit and Continue
Changes to static files and/or razor template files (.cshtml) are automatically updated without the need of a compilation step. Make the change, save and tap refresh in the browser to view the update.  

Modifications to code files require compiling and a restart of Kestrel within the container. After making the change, use CTRL + F5 to perform the process and start the application within the container. The Docker container is not rebuilt or stopped; using `docker ps` in the command line you can see that the original container is still running as of 10 minutes ago. 

```console
CONTAINER ID        IMAGE                          COMMAND               CREATED             STATUS              PORTS                   NAMES
3f240cf686c9        spboyer/hellodockertools:dev   "tail -f /dev/null"   10 minutes ago      Up 10 minutes       0.0.0.0:32769->80/tcp   hellodockertools_hellodockertools_1
```

### Publishing Docker images 
Once you have completed the develop and debug cycle of your application, the Visual Studio Tools for Docker will help you create the production image of your application. Change the debug dropdown to **Release** and build the application. The tooling will produce the image with the `:latest` tag which you can push to your private registry or Docker Hub. 

Using the `docker images` you can see the list of images.

```console
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
spboyer/hellodockertools   latest              8184ae38ba91        5 seconds ago       278.4 MB
spboyer/hellodockertools   dev                 0b6e2e44b3df        About an hour ago   268.9 MB
microsoft/aspnetcore       1.0.1               189ad4312ce7        5 days ago          268.9 MB
```

There may be an expectation for the production or release image to be smaller in size by comparison to the **dev** image, however through the use of the volume mapping; the debugger and application were actually being run from your local machine and not within the container. The **latest** image has packaged the entire application code needed to run the application on a host machine, therefore the delta is the size of your application code.
