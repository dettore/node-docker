# Learn Docker - DevOps with Node.js & Express

https://youtu.be/9zUHg7xjIqQ?si=gswNL70_bKpLYC17

Learn the core fundamentals of Docker by building a Node/Express app with a Mongo & Redis database.

We'll start off by keeping things simple with a single container, and gradually add more complexity to our app by integrating a Mongo container, and then finally adding in a redis database for authentication. 

We'll learn how to do things manually with the cli, then move on to docker compose. We'll focus on the challenges of moving from a development environment to a production environment. 

We'll deploy and Ubuntu VM as our production server, and utilize a container orchestrator like docker swarm to handle rolling updates.

## Project Initialization

Initialize package.json file

```
npm init -y
```

Install express

```
npm install express
```
### Simple Express App

Save as index.js

```
const express = require("express");

const app = express();

app.get("/". (req, res) => {
    res.send("<h2>Hi There</h2>")
});

const port = process.env.PORT || 3000;

app.listen(port, () => console.log(`listening on port ${port}`));
```

Start the app in the shell/terminal

```
node index.js
```

The app can be pulled up in the browser by going to localhost:3000

## Integrate Test App into Docker Container

The video assumes that Docker is already installed on your machine.

Log into Docker Hub and search for the official node image.  We will be creating a customer image off of the official image.

### Creating a custom Docker image

Dockerfile

```
FROM node:15
WORKDIR /app
COPY package.json .
RUN npm install
COPY . ./
EXPOSE 3000
CMD [ "node", "index.js" ]

```
Notes on the Dockerfile:

1. The example in the video uses version 15 of Node, because that was the current version at the the time.  
2. The WORKDIR for the container is set to /app
3. The package.json file is copied as a separate step because it most likely won't change again for the app, so this layer and the next will not need to be run again the next time that the container is recreated.
4. Have npm install the packages and dependencies from package.json
5. Copy the local app directory to the working directory in the container
6. Expose port 3000 from the container
7. The command and arguement to execute when the container is started

Create the custome image

```
docker build -t node-app-image .
```

Run the image to test it

-d runs the container in detached mode
--name names the container
-p 3000:3000 listen for and send traffic to port 3000 (left host, right container)

```
docker run -p 3000:3000 -d --name node-app node-app-image
```

If you want to log into the container

```
docker exec -it node-app bash
```
-it is for interactive
node-app is the name of the container
bash is the command to run inside of the container

```
root@ec1e681e052e:/app# ls -al
total 60
drwxr-xr-x 1 root root  4096 Oct 21 14:29 .
drwxr-xr-x 1 root root  4096 Oct 21 16:25 ..
-rw-r--r-- 1 root root   110 Oct 21 14:16 Dockerfile
-rw-r--r-- 1 root root  2394 Oct 21 14:29 README.md
-rw-r--r-- 1 root root   234 Oct 21 13:31 index.js
drwxr-xr-x 1 root root  4096 Oct 21 13:13 node_modules
-rw-r--r-- 1 root root 24276 Oct 21 13:13 package-lock.json
-rw-r--r-- 1 root root   270 Oct 21 13:13 package.json
```

There are files that were copied into this container that are not needed in the container.

Create a .dockerignore file

```
node_modules
Dockerfile
.dockerignore
.git
.gitignore
```

Remove the container when it is no longer needed

(-f forces the remove without first requireing a stop)

```
docker rm node-app -f
```

Rebuild the container while ignoring the unnecessary files
```
docker build -t node-app-image . 
docker run -p 3000:3000 -d --name node-app node-app-image
```

## Volumes
Volumes allow containers to have persitent data

### Bind Mount Volume
Syncing data from a specific mount point on the host

Add argument to `docker run` command line:
-v pathtofolderonlocalmachine:pathtofolderoncontainer

/Users/don/workspace/docker/node-docker/:/app

docker run -p -v /Users/don/workspace/docker/node-docker/:/app 3000:3000 -d --name node-app node-app-image

On Mac or Linux, you can use the following variable:

docker run -v $(pwd):/app -p 3000:3000 -d --name node-app node-app-image

Since the code was modified, the node process needs to be restarted (this is a requirement of Express).

Install nodemon on local dev machine (save as a dev dependency because it will not be needed when deploy is to production)

```
npm install nodemon --save-dev
```
Add scripts section to package.json

```
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
```

Change CMD in Dockerfile to the following and rebuild image:

```
CMD [ "npm", "run", "dev" ]
```

Show logs for a particular container (even if it has exited)

```
docker logs node-app
```

Set an anonymous mount to prevent the bind mount from overwriting the node_modules folder

```
docker run -v $(pwd):/app -v /app/node_modules -p 3000:3000 -d --name node-app node-app-image
```

Make bind mount as R/O

Add :ro at the end of the bind mount

```
docker run -v $(pwd):/app:ro -v /app/node_modules -p 3000:3000 -d --name node-app node-app-image
```

Running `docker volume ls` will show persitent volumes.  Adding anonymous volumes will build up the volumes over time.  Run `docker volume prune` to remove the orphaned volume.  Add -v to the `docker rm` command to delete the volume associated with a container when the container is deleted.

`docker rm node-app -fv`

## Using Environment Variables Inside of a Docker Container

Add PORT environment variable to the Dockerfile:

```
ENV PORT 3000
EXPOSE $PORT
```
(Note that EXPOSE is for documentation purposes only)

--env or -e to pass in environment variable on `docker run`

```
docker run -v $(pwd):/app:ro -v /app/node_modules --env PORT=4000 -p 3000:4000 -d --name node-app node-app-image
```

Run `printenv` in Linux to show the environment variables

Environment variables can be passed in via a file

`--env-file ./.env` if the file is called .env (standard convention)

## Docker Compose

Replaces the long `docker run` command line.   Allows for bringing up multiple containes to set up the dev environment (app, db, etc.).  The video uses `docker-compose.yml`, but the Docker Compose Specification states that the prefered name is now `compose.yaml`.  The previous file name is support for compatability.  

https://docs.docker.com/compose/compose-file/

The video also specifies a version in the file, and while it is supported for compatability reasons, it is no longer needed. 

docker-compose.yml
```
version: "3"
services:
  node-app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - ./:/app:ro
      - /app/node_modules
    environment:
      - PORT=3000
```

The video uses the command `docker-compose`, which still exists for compatability, but the new command is `docker compose`

`docker-compose --help` shows commands and options
`docker compose up --help` shows optons available for up

Starting and stopping the container - 

`docker-compose up -d` brings up the container and detaches it
`docker-compose down -v` brings down the container and deletes the volumes

Docker Compose does not detect if there was a change made for the image, so you need to force a build -

`docker-compose up -d --build`

## Development vs Production Configs

Set up `compose.yaml` file to have a separate set of commands for development and production.  You can have multiple `Dockerfile` and `compose.yaml`, one each for dev and prod.

`docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build`

`docker compose -f docker-compose.yml -f docker-compose.prod.yml down`

Add conditional statement in Dockerfile to run specific commands based on environment type.

```
ARG NODE_ENV
RUN if [ "$NODE_ENV" = "development" ]; \
        then npm install; \
        else npm install --only=production; \
        fi
```

## Adding Another Container

### Adding a MongoDB Container

Pull official MongoDB image from Docker Hub

`docker pull mongo`

Start a `mongo` server instance 

`$ docker run --name some-mongo -d mongo:tag`

Add Mongo container to docker-compose.yml

```
  mongo:
    image: mongo
    environment:
      - MONGO_INITDB_ROOT_USERNAME=don
      - MONGO_INITDB_ROOT_PASSWORD=mypassword
```

Bring the app and DB up

``

Video used `mongo` to get into Mongo, but the mondo shell was removed starting in MongoDB 6.0.  The replacement is `mongosh`.

`docker exec -it node-docker-mongo-1 bash`
`mongosh -p "don" -u "mypassword"`

Working with MongoDB

```
test> db
test
test> use mydb
switched to db mydb
mydb> show dbs
admin   100.00 KiB
config   60.00 KiB
local    72.00 KiB
mydb> db.books.insert({"name": "harry potter"})
DeprecationWarning: Collection.insert() is deprecated. Use insertOne, insertMany, or bulkWrite.
{
  acknowledged: true,
  insertedIds: { '0': ObjectId("65396bc34687b8788ad709e8") }
}
mydb> db.books.find()
[ { _id: ObjectId("65396bc34687b8788ad709e8"), name: 'harry potter' } ]
mydb> show dbs
admin   100.00 KiB
config   84.00 KiB
local    72.00 KiB
mydb     40.00 KiB
mydb> exit 
```

Tear down the container, then bring it back up

```
docker compose -f docker-compose.yml -f docker-compose.dev.yml down -
v
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```
Add Named Volume to docker-compose.yml for DB Data

```
    volumes:
      - mongo-db:/data/db

volumes:
  mongo-db:
```

### Set Up Express App to Connect to MongoDB Database

Install mongoose

(https://github.com/Automattic/mongoose)

`npm install mongoose`

1. Tear down the existing container (docker compose down)
2. Bring everything back up (docker compose up) with --build option
3. Import mongoose into the app `const mongoose = require('mongoose');`
4. Connect to the database

```
mongoose.connect(
    "mongodb://don:mypassword@mongo:27017/?authSource=admin")
    .then(() => console.log("successfully connected to DB"))
    .catch((e) => console.log(e));
```
Note that `mongo:27017` is the DNS entry in the docker network for these containers (mongo is the hostname, which is the service name).

## Define Environment Variables on a Config File

Create `config.js` file with a direcotry called `config` in the base directory.

```
module.exports = {
    MONGO_IP: process.env.MONGO_IP || "mongo",
    MONGO_PORT: process.env.MONGO_PORT || 27017,
    MONGO_USER: process.env.MONGO_USER,
    MONGO_PASSWORD: process.env.MONGO_PASSWORD
}
```
Makes migrating to Prod easier because the values are not hardcoded in the app.   Allows for more flexibility in the future should the values need to change.

Pass in environment variables via docker-compose

```
    environment:
      - NODE_ENV=development
      - MONGO_USER=don
      - MONGO_PASSWORD=mypassword
```
Need to rebuild the container in order to take effect

### Container Boot Up Order

Add `depends_on` to docker-compose in the app section

```
    depends_on:
      - mongo
```

Still need to add code in the app to make sure that it connects to the database or whatever other service it is dependent on.

## Deploying to Production

###Install Ubuntu Server on a Cloud provider or VM

###Install Docker on the Linux Server

https://docs.docker.com/engine/install/ubuntu/
https://get.docker.com (Follow steps under Usage)

Verify that install worked

```
don@boost:~$ docker --version
Docker version 24.0.7, build afdd53b
```

###Install Docker Compose

https://docs.docker.com/compose/install/linux/

The tutorial uses docker-compose, which is V1 and no longer supported.   It is still available for compatability reasons.  The prefered method is now `docker compose`.

Install Docker Compose

```
sudo apt-get update
sudo apt-get install docker-compose-plugin
```
```
don@boost:~$ docker compose version
Docker Compose version v2.21.0
```

Install docker-compose

`sudo apt install docker-compose`
```
don@boost:~$ docker-compose -v
docker-compose version 1.29.2, build unknown
```

## Moving to Production

If only the app is going to change, and if the containers are already up, you can specify which service to build. Adding `--no-deps` won't rebuild the dependent containers (since they probably didn't change).

`sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build --no-deps node-app`

You can also force a recreate with no deps as well

`sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --force-recreate --no-deps node-app`

### Pushing to Docker Hub

Create a new repository on Docker Hub (on website)
(May need to log in before next step: `node-docker-node-app`)
Rename the image that you want to push

```
docker image ls
REPOSITORY                       TAG             IMAGE ID       CREATED         SIZE
node-docker-node-app             latest          4f459a1fe853   2 days ago      918MB
```
`docker image tag node-docker-node-app dettore/node-app`

```
docker image ls
REPOSITORY                       TAG             IMAGE ID       CREATED         SIZE
dettore/node-app                 latest          4f459a1fe853   2 days ago      918MB
node-docker-node-app             latest          4f459a1fe853   2 days ago      918MB
```

Push image to docker hub 

`docker push dettore/node-app`

Tell Docker Compose to pull the image from Docker Hub

`image: dettore/node-app`

Make code changes and then build locally (for Prod)

`docker compose -f docker-compose.yml -f docker-compose.prod.yml build node-app`

`docker compose -f docker-compose.yml -f docker-compose.prod.yml push node-app`

`sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml pull`

`sudo docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --no-deps node-app`

## Automating Docker Container Base Image Updates - Watchtower

Periodically checks the Docker Hub repository for updates to images.  

This may not be something that you want to do in Production, but it has been provided as an example for automating part of the production deployment process.

https://github.com/containrrr/watchtower

The full documentation is available at https://containrrr.dev/watchtower.

`sudo docker run -d --name watchtower -e WATCHTOWER_TRACE=true -e WATCHTOWER_DEBUG=true -e WATCHTOWER_POLL_INTERVAL=50 -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower app-node-app-1`

To remove and delete the contaomer:  `sudo docker rm watchtower -f`

## Container Orchastrator

Docker Swarm is a built-in container orchastrator

"rolling updates" (minimal downtime for app)

(BTW, Kubernetes is a container orchastrator)

Docker Swarm contains two type of nodes:  Manager Node and Worker Node

Swarm is installed by default with Docker, but it is inactive:

```
sudo docker info
...
Swarm: inactive
...
```
`sudo docker swarm init`

It looks like to default is to set the current node as manager, if an IP addr was not specified.   IP add can be specified with:

`sudo docker swarm init --advertise-addr 192.168.40.94`

```
don@boost:~/app$ sudo docker swarm init --advertise-addr 192.168.40.94
Swarm initialized: current node (iy184oulngfz3u0gjsxnhf5m3) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4ab34pd6izg81ejxnncx1xjv41sodv8bwprfzhl3mwg5k7oxfe-6ml3alrrz7hseldnafld4axrk 192.168.40.94:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

If a node is added as a manager, it is also a worker by default.

Use `docker service` to manage Swarm services

Docker Swarm options can be set in the Docker Compose file under a `deploy:` section.  These are only set in the Prod version since Swarm is not needed in Dev.

```
  node-app:
    deploy:
      replicas: 8
      restart_policy:
        condition: any
      update_config:
        parallelism: 2
        delay: 15s
```
Use `docker stack` on prod server to deploy

`sudo docker stack deploy -c docker-compose.yml -c docker-compose.prod.yml myapp`