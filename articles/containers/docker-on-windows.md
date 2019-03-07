# Docker for Windows

## Docker Setup on Windows 10

1. Press the Windows key then type ```Windows Features``` and select ```Turn Windows features on or off```
2. Check the box for ```Containers```, and ```Hyper-V``` then select ```OK```
3. Restart the computer for changes to take effect
4. Download and install [Docker for Windows](https://store.docker.com/editions/community/docker-ce-desktop-windows).  Will require logging into or registering with Docker (free)

## Running Angular Application within a Linux Container

Located at ```src/server``` is are both ```index.js``` and ```package.json``` files needed to serve up any static website.  The defaults are to server up index.html on port 80, but this can easily be changed in ```index.js```

1. Copy ```src/server``` folder to your Angular project's ```src``` folder
2. Edit ```angular.json -> build -> assets``` section adding the line ```{ "glob": "**/*", "input": "src/server/", "output": "/" },``` for including the Node server when building the application
3. Build the Angular project (```ng build```).  Will build development environment if not specified
4. Build the Docker container using the command ```docker build --rm -f "Dockerfile" -t paulgilchrist/angular-template:latest .```
5. Optionally run the container using the command ```docker run -d -p 80:3000 paulgilchrist/angular-template```
6. Optionally test the container in your browser at ```http://localhost```
7. Optionally push the container to the Docker Hub registry using the command ```docker push paulgilchrist/angular-template```

### Other useful Docker commands:

* View all images ```docker images```
* Remove single image - ```docker rmi <imageId>```
* View running containers ```docker ps```
* View all containers ```docker ps -a```
* Stop a container - ```# docker rm -f <containerID>```
* Inspect a running image's configuration - ```docker inspect <containerId>```
  * Useful for determining container IP address and other helpful details
* Purging all unused or dangling images, containers, volumes, and networks - ```docker system prune```
* If it is ever required to remove all stopped containers, it can be done with the command ```FOR /f "tokens=*" %i IN ('docker ps -a -q') DO docker rm %i```
