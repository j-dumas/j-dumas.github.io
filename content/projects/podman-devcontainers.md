+++
title = "Podman Development Containers"
+++

> Part of my [dotfiles](https://github.com/j-dumas/dotfiles)

## Summary
I am adding a base Containerfile (Dockerfile) and a script to create Podman containers with images based on the base Containerfile to create development environments as containers.
It came from wanting DevContainers without having to use VSCode (I know there are Neovim features, but still...). 
I also looked at Distrobox, but I wanted more isolation for my home directory.

## Goal
The goal of this project is to learn containerization and Podman, but mostly have a working and clean development environment for developing other projects without installing dependencies on my host system.
I also wanted the possibility to have a base environment with my tools that I can add to for specific projects.

## Content
Here is the base Containerfile based on Arch Linux.
I mostly install needed packages and create a user and a nice prompt.
```Dockerfile
FROM archlinux:latest 
LABEL Description="Base development env" 

## Installed needed development packages
RUN pacman -Syu --noconfirm \
    base-devel \
    zsh \
    neovim \
    git \
    wget 

RUN pacman -Scc --noconfirm 

## Create non root user
RUN useradd -m -s /usr/bin/zsh jdumas
USER jdumas

## Initialize folders so they are not root owned on mount
RUN mkdir /home/jdumas/.local && mkdir /home/jdumas/.config

ENV HOME /home/jdumas
ENV PROMPT="%F{##a6e3a1}%n%f@%F{##89b4fa}%m%f:%F{##fab387}%~%f $ "

WORKDIR /home/jdumas/src
CMD ["/usr/bin/zsh"]
```

Then, to create the container, I use the following script.
It takes a flag (`-n`) to tell the script that we want to use the base image for the container (otherwise we need a new Containerfile for dependencies additions).
It also takes a name for the container and the optional path to the project folder to mount it in the container to work on it.
It basically parses arguments, creates images and creates a container with the appropriate volumes.
```bash
##!/usr/bin/env bash

BASE_IMG_NAME="devenv"
NO_NEW_IMG=false

## Order of parameters :
##   1. flag to use base img for the container
##   2. name
##   3. path to project (to share folder)
usage() {
    echo "Usage: devenv [-n for no new image] <container name> [project path]"
}

## Parse argument -n and -p
while getopts "np:" opt; do 
    case ${opt} in 
        n) NO_NEW_IMG=true;;
        p) PORTS="127.0.0.1:$OPTARG";; ## Stay on localhost
    esac
done
shift $(($OPTIND - 1))

NAME=$1
PROJECT_PATH=$(realpath ${2:-$(pwd)})

if [ -z $NAME ]; then
    usage
    exit 1
fi

## Create base image
if ! podman image exists $BASE_IMG_NAME; then
    podman build -f ~/BaseEnvContainerfile -t devenv .
fi

## Create other image if needed
if [ $NO_NEW_IMG = true ]; then
    IMG_NAME=$BASE_IMG_NAME
else 
    IMG_NAME=$NAME
    podman build -f $PROJECT_PATH/EnvContainerfile -t $NAME $PROJECT_PATH
fi

## Create container with correct image, volumes and namespace
CMD="podman start -t --name $NAME \
    --userns=keep-id \
    -v $HOME/.config/nvim:/home/jdumas/.config/nvim \
    -v $HOME/.local/dev/nvim-container-share:/home/jdumas/.local/share/nvim \
    -v $PROJECT_PATH:/home/jdumas/src \
    -h $NAME"

if [[ -n "$PORTS" ]]; then
  CMD="$CMD -p $PORTS"
fi

CMD="$CMD $IMG_NAME"
eval $CMD
```

For example, a Containerfile using the base would be called `EnvContainerfile` and would look like :
```Dockerfile
FROM devenv
LABEL Description="Game Theory env"

USER root

RUN pacman -Syu --noconfirm \
    graphviz

RUN pacman -Scc --noconfirm 

USER jdumas
```

This is the Containerfile for a game theory project using Graphviz to display trees.
I would create the container by calling the script like this from the project directory :
```bash
devenv tree_viewer .
```

I can then start it with
```bash
podman start -ia tree_viewer
```

I can also connect multiple times with
```bash
podman exec -it mycontainer /bin/zsh
```
