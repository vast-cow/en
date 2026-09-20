---
title: "How to Use a Debian Trixie Docker Image with Wine"
description: "This Dockerfile creates a container based on Debian Trixie with Wine installed. The main purpose is..."
pubDatetime: 2026-02-12T11:45:52.867Z
---

This Dockerfile creates a container based on Debian Trixie with Wine installed. The main purpose is to run Windows applications inside a Docker container using Wine.



## Building the Docker Image

First, save the content as a file named `Dockerfile`.

Then build the image with:

```bash
docker build -t debian-wine .
```

This command creates a Docker image named `debian-wine` that includes Wine Stable from the official WineHQ repository.



## Running the Container

Start the container interactively:

```bash
docker run -it --rm debian-wine
```

You will enter a Bash shell inside the container because the default command is:

```dockerfile
CMD ["bash"]
```

The working directory is `/root`.



## Running Windows Applications

Inside the container, you can use Wine to run Windows executables:

```bash
wine your-program.exe
```

If needed, you can copy files into the container:

```bash
docker run -it --rm -v $(pwd):/work debian-wine
```

Then access them inside the container:

```bash
cd /work
wine your-program.exe
```

This allows you to execute Windows applications stored on your host machine.



## Key Points of Usage

* The container includes both 64-bit and 32-bit support (i386 architecture enabled).
* Wine Stable is installed from WineHQ.
* The environment is non-interactive for smooth automated builds.
* The image is cleaned to remain lightweight.

This setup provides a clean, isolated environment for running Windows software on a Linux system using Docker and Wine.


```
FROM debian:trixie 
 
ENV DEBIAN_FRONTEND=noninteractive 
 
# Base packages 
RUN apt-get update && \ 
    apt-get install -y --no-install-recommends \ 
        ca-certificates \ 
        wget \ 
        gnupg \ 
        && rm -rf /var/lib/apt/lists/* 
 
# Enable i386 architecture (required for Wine) 
RUN dpkg --add-architecture i386 
 
# WineHQ keyring 
RUN mkdir -pm755 /etc/apt/keyrings && \ 
    wget -O /etc/apt/keyrings/winehq-archive.key \ 
      https://dl.winehq.org/wine-builds/winehq.key 
 
# WineHQ repository (Trixie) 
RUN wget -NP /etc/apt/sources.list.d/ \ 
      https://dl.winehq.org/wine-builds/debian/dists/trixie/winehq-trixie.sources 
 
# Install Wine 
RUN apt-get update && \ 
    apt-get install -y --install-recommends \ 
        winehq-stable && \ 
    rm -rf /var/lib/apt/lists/* 
 
WORKDIR /root 
CMD ["bash"] 
```
