# Docker 

## 1. Initial Interaction with docker images and container.
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

## Optimize your Docker File.
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

## Publish your image in DockerHub.

- Make repo in Dockerhub.
- Build image and name it `[username]/[repo]` for unique image.
- docker push `imageName`.

ELSE 

- Build image with your desired name say `my-app`.
- `docker tag my-app hiimvikash/node-application` : here your image is given a unique tag `[username]/[repo]`.
- or `docker tag my-app hiimvikash/node-application:v1`

- `docker push hiimvikash/node-application:v1`

## Multistage builds.

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

