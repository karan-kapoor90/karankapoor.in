---
layout: post
categories: [Workshops]
comments: true
excerpt: A short and sweet getting started with Docker guide.
title: Docker Workshop
tags: [docker, cloud native, containers]
description: A short and sweet getting started with Docker guide.
---

This is a very basic, 20 minute quick guide to running your first docker container running an application. 

## Prerequisites
- [Docker desktop downloaded and installed](https://www.docker.com/products/docker-desktop) on your machine. 
- [Git command line downloaded and installed](https://git-scm.com/downloads) on your machine
- A [github user](https://github.com/join) account (optional)
- A [docker hub account](https://hub.docker.com/signup)

### Verify your installation

1. Start the docker service on your machine and check the docker version.

    ```bash
    docker -v
    ```

2. Run your first hello world container

    ```bash
    docker run --name hello hello-world
    ```

### Playing with your installation

3. Ok, so your container can say hello, can you run some other commands inside it?

    ```bash
    docker run busybox uptime
    ```

### Don't try this at 'host'

4. So, if it's an OS, can I log into it on the command line and actually do stuff? Sure!!

    ```bash
    docker run -it busybox sh
    ```

    Now try 

    ```bash
    whoami
    ```

    Surprise surprise!! You're root!! But wait, if I'm root now, can't I do anything? I mean *ANYTHING* !! Let's try something crazy!! 

    > PS: Make sure you're still inside the container, you DON'T want to do this on your machine.



    ```bash
    ls 
    uptime
    # ok, so commands work! Also..
    # I see bin.. evil laugh.. let me wreak havoc, and delete it!! I'm ROOT!!
    rm -rf bin
    # and now..
    ls
    uptime
    # Honey I blew up the machine!! Did I break my host machine too?
    exit
    ```

    However, once outside of the container, try `ls` and `uptime` again (if you're on a linux/ mac that is..), and they're still working..

    Voila! Docker isolation in action!!

### Serving a static webpage

5. I have an html page, and I want to serve it using an nginx server..

    ```bash
    # Creating an simple webpage that says Hello!!
    echo <html><head></head><body>Hello!!</body></html> > index.html

    # Installing and enabling an httpd server in one single command
    docker run -dit --name my-web-server -p 8080:80 -v "$PWD":/usr/local/apache2/htdocs/ httpd:2.4

    # Launch your browser and check your machine's port 8080 (mac/ linux)
    open "http://localhost:8080"   # for mac
    explorer "http://localhost:8080"   # for windows

    ```

6. Cleanup running containers

    ```bash
    docker ps 

    # Notice the status of the container named my-web-server. Stop it
    docker stop my-web-server

    # Remove the container
    docker container rm my-web-server
    ```

### Dockerizing a node.js app

6. Let's now do something a little more real world!! Let's get our hands on a very simple node app, create a docker container for it, push it to a repo, and learn how to pull and run it. 

7. Login to github and pull the following repository:

    ```bash
    git clone https://github.com/karan-kapoor90/docker-workshop.git

    # consider running the application locally if you have node installed on your machine
    npm install
    node index.js
    # But the happy scenario, in case you don't have nodejs installed on your machine, docker to the rescue
    ```

8. Explore the code.. as a shortcut, consider reading the readme file on the [git repository](https://github.com/karan-kapoor90/docker-workshop).

9. Build a docker container for your code. PS: The following step creates a docker image which
    - Is based on a nodejs base image
    - Takes a copy of your code's dependencies (that's the package.json file for a nodejs app)
    - Downloads all those dependencies into a folder
    - Copies your code into the same folder
    - Declares the command that is used to run your code (`node index.js` in this case.)

    > PS: My dockerhub username is karankapoor

    ```bash
    export DOCKER_USERNAME=<your-dockerhub-username>
    # building your container locally
    docker build -t $DOCKER_USERNAME/docker-workshop:latest -f Dockerfile .
    ```
    > All the steps defined above, and more, such as setting environment variables, creating an image that depends on the output of multiple such images (aka multistage images) etc., is all defined in the `Dockerfile` file in the code you just checked out from github.


10. Run the container locally, exposing the app on port 30000 of your local host machine

    ```bash
    docker run -d -p 30000:3000 --name my-app $DOCKER_USERNAME/docker-workshop:latest

    # inspect the docker container running in the background
    docker ps
    ```

11. Explore your shiny new docker app in the browser

    ```bash
    http://localhost:30000/
    http://localhost:30000/welcome.html
    http://localhost:30000/hello   # a GET endpoint that responds with some JSON content
    ```

12. Clean up

    ```bash
    docker container stop $(docker container ls -aq)   # Stop all running containers
    ```


### Pushing your first image to dockerhub


13. Your first container for the world to see

    ```bash
    docker login   # it will ask you for your docker registry credentials. Since we're using dockerhub, no need to provide a registry URL
    docker push $DOCKER_USERNAME/docker-workshop:latest   # this will push your docker container to docker hub, kind of like a github for docker containers.
    ```

Now you're a champ! But why stop here? If you installed docker desktop for your platform, most likely, you also have a local distribution of kubernetes running on your machine.

For more info on why you should consider orchestrating your container using kubernetes, and a basic how-to, follow the guide [here](kubernetes-workshop).