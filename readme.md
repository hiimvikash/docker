# Docker 

## 1. Initial Interaction with Docker images and containers.
- [Watch Section 1 > video 2 & 3](https://drive.google.com/drive/folders/18IndpFNBZJ3JfLn9AWnvie459h6jIoeI?usp=sharing)
- [Containers vs. virtual machines](https://www.atlassian.com/microservices/cloud-computing/containers-vs-vms)
- Containers are like independent isolated machine.
- Docker GUI download.
- `docker --version`
- IMAGE -> CONTAINER1, CONTAINER2 ....
- Run `docker COMMAND --help` for more information on a command.
- `docker ps` : list running containers.
- `docker ps -a` : list all the containers.
- `docker images` : list all images.
- `docker pull busybox` : pulls the image from Dockerhub on your local machine.
- `docker kill e456abc` : kill particular container.
- `docker container rm e456abc a456ej45 l456ej45` : remove one more container from local machine.
- `docker image rm alpine` : remove alpine image from local.
- `docker run -it --name myContainer ubuntu` : spin a container(if no --name then random name is provided to a container) from ubuntu image as `interactive-terminal`.
- `docker image inspect ubuntu`
    - every image has it's entry point, default entry point is `/bin/bash` for ubuntu.
    - `docker run ubuntu` → Runs, finds no command, and exits immediately.
    - `docker run -it ubuntu` → Runs an interactive shell, so it keeps running.
    - `docker run -it ubuntu ls` → This will run `ls` and exit. 

## 2. Write your own docker file.

- Let's say you have a simple express backend listening on PORT = 3000.

- Now What are the things you may need to run same express server in different laptop ?
    - Nodejs 20.3
    - npm 10.2.3
    - backend source code.

```docker
FROM ubuntu #this is the base image

#installing nodejs
RUN apt-get update
RUN apt install -y curl
RUN curl -sL https://deb.nodesource.com/setup_16.x -o /tmp/nodesource_setup.sh
RUN bash /tmp/nodesource_setup.sh
RUN apt install -y nodejs

# Copying code files, destination directory are dynamically created while copying
COPY index.js /home/app/index.js
COPY package-lock.json  /home/app/package-lock.json
COPY package.json  /home/app/package.json

WORKDIR /home/app/

RUN npm install
```

- `docker build -t my-app .`

**1️⃣ Why does running the container directly open the bash terminal?**

Your Dockerfile does not specify a default command (CMD) or entry point (ENTRYPOINT).

- When you run a container from this image, Docker doesn't know what to execute.
- Since your base image is ubuntu, it defaults to starting a Bash shell (/bin/bash).
- if you want it to run your index.js file, you need to explicitly specify it: `CMD ["node", "index.js"]`

**2️⃣ What is the significance of `WORKDIR /home/app/`?**

- **What does it do? →** It sets the working directory for any following `RUN`, `CMD`, `ENTRYPOINT`, or `COPY` commands.

- **Why is it useful? →** Instead of writing: `RUN cd /home/app && npm install`
You can just set `WORKDIR` once and write: `RUN npm install`

*Effect: Any commands after WORKDIR will execute inside `/home/app/`.*

3️⃣ Difference between `CMD` vs `RUN` ?

**RUN :**
1. executes command while building the image.
2. once completed result is saved in final image.
3. so if you create multiple container from this image, they will already have the effect of `RUN`.

**CMD :**
1. CMD is used to specify the default command when the container runs (after docker run).
2. It does not execute at build time.
3. It can be overridden when starting the container.

    - If I add this dockerFile above : `CMD ["node", "index.js"]`

    - Effect:

    - When you run docker run my-image, it will execute node index.js inside the container.
    - If you run `docker run my-image bash`, it overrides CMD, and bash runs instead of node index.js.
4. If you have multiple CMD in your dockerfile, then only last CMD will be executed.

## 3. Optimize your Docker File.
- We want our image as small as possible so it's easy to share, always use a light weight base image.

- Each line in Dockerfile is layer, if you make change in `layer 5` in your dockerfile then all the lines(layers) above line 5 are cached, build starts from `layer5 to end`
- set your WORKDIR @ start so that commands like `COPY`, `RUN` on the particular directory. 

```docker
FROM node:20.17.0-alpine3.19

WORKDIR /home/app

COPY package*.json •
RUN npm install

COPY index.js •
COPY Dockerfile •

CMD [npm, start]
```


- **PORT maping :** `docker run -it -p 3000:8000 my-app`

- If you give `EXPOSE 8000` in your dockerfile, then you can also run `docker run -it -P my-app`, this will automatically do random port mapping.

- `docker run -it -P --rm my-app` : removes the container from local machine when stoped.

- `docker run -itd -P --rm my-app` : detach mode.

## 4. Publish your image in DockerHub.

- Make repo in Dockerhub.
- Build image and name it `[username]/[repo]` for unique image.
- docker push `imageName`.

ELSE 

- Build image with your desired name say `my-app`.
- `docker tag my-app hiimvikash/node-application` : here your image is given a unique tag `[username]/[repo]`.
- or `docker tag my-app hiimvikash/node-application:v1`

- `docker push hiimvikash/node-application:v1`

## 5. Multistage builds.

When you have a typescript application :
- At the end of the day we just compile typescript file which results in javascript files and at the end we always run a javascript file.

- then there are many dependencies related to typescript which will be usefull only while building that project like @types/express, typescript and etc.

- So here in multistage builds we first build our project while creating image then we make final image with `dist` folder and all the typescript related dependencies are omitted, node modules are freed with unneceesary typescript related modules.

- whatever is the last stage while building image, that will be onl shared with every one. 


```docker
FROM node:20.17.0-alpine3.19 as base

# Stage 1: Build Stuff
FROM base as builder
WORKDIR /home/build
COPY package*.json • 
COPY tsconfig.json •
RUN npm install
COPY src/ src/
RUN npm run build

# Stage 2: Runner
FROM base as runner
WORKDIR /home/app
# after copying from builder stage other files are deleted.
COPY --from=builder /home/build/dist dist/ 
COPY --from=builder /home/build/package*json • 
RUN npm install —omit=dev
CMD ⁠[ "npm" "start" ]
```
**Enviroment Variables :** `docker run -it -p 4000:4000 -e PORT=4000 mytsapp`
- -e PORT=3000 injects the environment variable PORT=3000 into the container.
- Inside the container, process.env.PORT gets the value from the injected environment variable.
- Your app reads PORT and starts on port 3000.

## 6. Docker Networking
### WHY Are We Studying Docker Networking ?

When we run applications in Docker containers, they need to communicate with each other and the outside world. Networking in Docker ensures:

- Containers can talk to each other (e.g., a frontend app can communicate with a backend API).
- Containers can communicate with external services (e.g., databases, APIs, or cloud services).
- Security and isolation – We control who can access what in a Docker environment.
- Port Mapping & Exposure – Making container services accessible to the host system and the internet.

🔹 Example Scenario:Imagine you are building a web app that consists of:

- **Frontend (React)** running in one container
- **Backend (Node.js/Express)** running in another container
- **Database (PostgreSQL)** running in another container

Without proper networking, these containers cannot talk to each other! 🚧

### Some Experiments :
![image](https://github.com/user-attachments/assets/908eddc5-084e-4270-a94e-a0d78134e6c7)

- `docker run -itd --name=container_one --rm busybox`
- Run a container (assign a name) in detached mode.
- `docker network ls` : Here you will see all the docker network provided by docker engine.
- `docker network inspect bridge` : this will show you the cofig of bridge network and also all the containers running in bridge network, here you will find container_one in this bridge network having ip : 172.17.0.2 which will be diff from your HOST machine ip address.NOW SPIN UP another container with different name (container_two),you will find this 2nd container is also connected with same bridge network with ip : 172.17.0.3.

- NOW WILL YOU BE ABLE TO PING container_one from container_two ?
- YES, `docker exec container_two ping 172.17.0.2` will work.
- MACHINE IN THE SAME NETWORK CAN COMMUNICATE WITH EACH OTHER PRIVATELY, THEY DON'T NEED TO EXPOSE.


#### EXAMPLE :-
- nodejs container running @ port 8000 and it is exposed & mapped with host machine.
- but the containers like postgress and redis services can run on same network so that nodejs container can connect with it privately, without exposing the ports to host machine.
- **When you start Docker, a default bridge network (also called bridge) is created automatically, and newly-started containers connect to it unless otherwise specified.**
![image](https://github.com/user-attachments/assets/3b152979-af48-400f-bf35-299c5c963d80)




### WHAT Is Docker Networking?

Docker provides networking features that allow containers to communicate securely and efficiently.

**🔹 Key Points About Docker Networking:**

- Every container has a network interface with an IP address, a gateway, and a routing table.💁🏻
- Containers can communicate with other containers or with the external world.
- Different network drivers (bridge, host, overlay, etc.) provide different networking capabilities.You can create custom networks to define how containers communicate.

Every container has a network interface with an IP address, a gateway, and a routing table.

#### 💁🏻 means : Every Container Has a Network Interface

When you create a container in Docker, it is like creating a tiny computer inside your actual computer. Just like a real computer, it needs:

- **An IP Address →** A unique identifier for the container in the network (like 192.168.1.100).
- **A Gateway →** The “door” the container uses to communicate outside its own network.
- **A Routing Table →** A set of rules that decide where the container sends its network traffic.

Think of It Like Your Home WiFi 🛜
- Your Laptop = A Container
- Your Router = The Gateway
- Your Home Network = Docker Network

Just like your laptop gets an IP address from the WiFi router, every container gets an IP from Docker’s network and follows rules for sending/receiving data. 🚀

### How Does It Work?

When you start a container, Docker assigns it an IP address (by default, inside a private Docker network). The container can use this IP to:

Talk to other containers on the same network.Send data to the internet through the gateway.Follow network rules defined in the routing table to decide where traffic should go.

**When you run a database inside a container, it is isolated by default. Your backend container cannot access it unless both are in the same network.**

📍 Here we are creating new network and running our backend and DB in the newly created network so our backend container can connect with local DB running in a diff container without actually exposing PORT to the HOST.


By creating a custom network (my-network) and connecting both containers to it:

- They can communicate using container names instead of IPs.
- No need to expose ports to the host machine (-p 3306:3306 is not required).
- Better security & isolation → Only containers inside my-network can access the database.

So yes, that's why we create a new network instead of using the default bridge network! 🚀





1️⃣ Bridge Network (Default)

📌 What is it?

If you don’t specify a network, Docker creates a bridge network automatically.It allows multiple containers on the same machine to communicate with each other.

📌 When to use?

When running multiple containers that need to talk to each other, like a backend app and a database.

📌 Example:

- `docker network create my-bridge-network`  # Create a bridge network
- `docker run --network=my-bridge-network -d --name=backend my-backend-image`
- `docker run --network=my-bridge-network -d --name=db my-database-image`


👉 Now, backend can connect to db using its name instead of an IP!

### 🎯 Goal - Creating a custom network.
![image](https://github.com/user-attachments/assets/70a3acba-ae2e-424b-ac97-688efc1e8d60)


Here you will see we can ping other container using their names : this is called DNS resolution in User defined network.👀
![image](https://github.com/user-attachments/assets/d190b0b4-f80f-48fd-af23-c454b84348ff)


- ✅ Create a custom Docker network
- ✅ Run a PostgreSQL database container
- ✅ Run a Node.js backend container
- ✅ Connect the backend to the database using the container name instead of an IP

**Step 1: Create a Custom Docker Network**

`docker network create my-bridge-network`


This creates a user-defined bridge network, allowing containers to communicate by name.

**Step 2: Run a PostgreSQL Database Container**

```docker
docker run -d \
  --name my-postgres \
  --network=my-bridge-network \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  postgres
```


*✅ This starts a PostgreSQL database named my-postgres inside the my-bridge-network.✅ The database listens inside the container on port 5432.*

**Step 3: Run a Node.js Backend Container**

```docker
docker run -d \
  --name my-backend \
  --network=my-bridge-network \
  -e DATABASE_HOST=my-postgres \
  -e DATABASE_USER=admin \
  -e DATABASE_PASSWORD=secret \
  -e DATABASE_NAME=mydb \
  my-backend-image
```


*✅ The backend can connect to PostgreSQL using my-postgres instead of an IP.✅ This works because both containers exist in the same network.*

**Step 4: Verify Connection**

Run the following command to inspect the network:

`docker network inspect my-bridge-network`


You'll see that both containers are listed inside this network.

To test if the backend can reach the database, exec into the backend container and try to ping my-postgres:

`docker exec -it my-backend ping my-postgres`


If everything is set up correctly, you'll see successful ping responses. 🚀


🌟 Recap

- ✔ We created a Docker bridge network
- ✔ We started a PostgreSQL container inside the network
- ✔ We ran a Node.js backend, connecting to PostgreSQL using its name (my-postgres)

👉 This is how Docker networking allows containers to talk to each other without using IP addresses! 🚀

 

### Default Bridge VS User-Defined Bridge🫣

#### 1️⃣ User-Defined Bridges Provide Automatic DNS Resolution

##### 📌 What This Means

In a user-defined bridge network, containers can find each other by name (like db or backend).In the default bridge network, containers can only talk using IP addresses, unless you use the old --link option.

#### User-Defined Bridge (Easier)👏🏻

If we create a custom bridge network: `docker network create my-custom-network`


Then, start two containers (web and db) inside this network:

- `docker run -d --name db --network=my-custom-network postgres`
- `docker run -d --name web --network=my-custom-network nginx`


*➡️ Now, the web container can connect to db using its name (db). No need for an IP! 🚀*

#### Default Bridge (Harder)👏🏻

If we don't specify a network, they go into the default bridge:

- `docker run -d --name db postgres`
- `docker run -d --name web nginx`


*➡️ Now, web can't automatically find db by name. It has to use the IP address, which changes on restart.*

#### 2️⃣ User-Defined Bridges Provide Better Isolation

##### 📌 What This Means

If you don't specify a network, all containers go into the default bridge network.This means any container can potentially talk to any other container, which is a security risk.A user-defined bridge isolates services, meaning only containers inside that bridge can communicate.

#### User-Defined Bridge (Safer)👏🏻

If we create a network for backend services only:

- `docker network create backend-network`
- `docker run -d --name db --network=backend-network postgres`
- `docker run -d --name backend --network=backend-network node`


*➡️ Now, frontend cannot talk to db, ensuring better security. 🔒*

#### Default Bridge (Risky)👏🏻

If we run three services without defining a network:

- `docker run -d --name db postgres`
- `docker run -d --name backend node`
- `docker run -d --name frontend nginx`


*➡️ Here, frontend can talk to db, even if it shouldn’t because here all the critical services like DB and backend will be exposed in default network where other containers (which has no relation will also be able to ping our backend) 😨*

#### 3️⃣ Containers Can Be Attached/Detached from User-Defined Networks on the Fly

##### 📌 What This Means

In a user-defined network, you can attach or remove a running container without stopping it.In the default bridge network, you must restart the container to change its network.



#### With a User-Defined Bridge (Flexible)

`docker network create my-network`
`docker run -d --name backend --network=my-network node`


*➡️ If we want to move backend to another network, we can do it while it's running:🔁*

- `docker network disconnect my-network backend`
- `docker network connect another-network backend`


#### With the Default Bridge (Less Flexible)

If a container is in the default bridge:

`docker run -d --name backend node`


*➡️ We can’t change its network without stopping it:*

- `docker stop backend`
- `docker rm backend`
- `docker run -d --name backend --network=my-new-network node`


This is much more work! 😓

#### 4️⃣ Each User-Defined Network Creates a Configurable Bridge

##### 📌 What This Means

The default bridge network has one shared configuration for all containers.A user-defined network can have custom settings (like firewall rules and MTU size).





#### 5️⃣ User-Defined Networks Expose All Ports to Each Other

#### 📌 What This Means

Containers in the same user-defined network can access each other's ports without extra configuration.But to expose a container outside its network, you need to publish ports (-p).


Same Network (Can Access Each Other's Ports)

- `docker network create my-network`
- `docker run -d --name db --network=my-network -p 5432:5432 postgres`
- `docker run -d --name backend --network=my-network -e DATABASE_HOST=db node`


*➡️ backend can access db:5432 without extra steps. 🎉*

- **Different Networks (Needs -p)**

If backend is in a different network:

- `docker run -d --name db -p 5432:5432 postgres`
- `docker network create another-network`
- `docker run -d --name backend --network=another-network node`


*➡️ Now, backend can’t reach db, unless we explicitly connect it to db’s network.*



🎯 Final Thoughts

If you're running multiple services, always use a user-defined bridge for better networking.The default bridge is only good for quick, simple container setups.With Docker Compose, user-defined networks are automatically managed, making life even easier.


## 7. Docker Volume 

### Run a ubuntu container, create files and directory using interactive terminal.
- Now will you be able to persist those file if you re-run the container ?
- Can you access particular directory of your HOST machine from the container ?

The answer based on our current knowledge is NO, because running a container means running a small machine in full isolation which will have it's own file system.

### The concept of Docker Volume helps container to use HOST machine HardDisk.

#### Way 1 : Volume mounting.
- Here we mount (link) **particular directory of HOST** to **particular directory of container** while running container.
- so any file change from anyone(HOST or CONTAINER) will be visible by both..basically we have single source of truth.
- `docker run -it --rm -v /Users/piyushgarg/Downloads/shared-folder:/home/ubuntu/piyush ubuntu`
  ![image](https://github.com/user-attachments/assets/6dd5871a-d513-4c94-908e-e09b2841c22d)

#### Way 2 : Volume by Docker 
- Here we create a actual VOLUME using docker and give it a name.
- any container can come and use this VOLUME to store their files.
- `docker volume create custom_data`
- `docker run -it --rm -v custom_data:/server ubuntu`
- make files 
- `docker run -it --rm -v custom_data:/server busybox` : Here you will be able to acces same files by attaching default volumes.


## 8. Docker compose 
- Docker Compose is a tool that helps you define and run multi-container Docker applications using a single YAML file (docker-compose.yml).
- Instead of running multiple docker run commands for different containers, Docker Compose allows you to define everything in one place and start everything with a single command.
- Here you specify specific version of services.
- All the services are running on closed custom network.
- `docker compose up -d`
- `npm run build && npm run start`
![image](https://github.com/user-attachments/assets/fc0649f1-7621-47bd-b385-799e1e887598)

### Networking in Docker Compose (we don't use much)
- Here we can create our custom network and assign services to run our user defined network.
![image](https://github.com/user-attachments/assets/8d0da2b8-e42e-43a8-8cb3-f944c07d859f)

### Volumes in Docker compose
![image](https://github.com/user-attachments/assets/3320b255-5b13-4379-9e5c-33482a5705b3)

### Custom Docker service : How can I bring my backend in docker compose file ? 
- Here my backend, db, redis are running in same network and if u remember in our backend file we are accesing (localhost:5432 for postgress and localhost:6379 for redis).
- So this will result in fail connection because if you are running your backend as container then localhost connection to other services doesn't make sense.
- Earlier it was localhost because your backend was running on HOST machine and postgress and redis were PORT mapped and EXPOSED to HOST.
![image](https://github.com/user-attachments/assets/f805e6df-72e5-4ea2-88cf-a35f6610379e)
![image](https://github.com/user-attachments/assets/a64e09e6-e17d-4b05-af12-0ee7a1b01e07)

#### Solution is just change backend code
- Here redis is connecting in `redis://redis:6379`
- postgress `host=db` & `port=5432`
![image](https://github.com/user-attachments/assets/cc560f86-1137-48ce-ba3c-b399ca092c9e)
- You can remove PORT MAPPING for redis and db service from docker-compose.yml file and add PORT mapping for backend service.








