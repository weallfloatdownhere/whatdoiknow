# Docker documentation
Here, you will find anything related to `Docker`. We will go over the processes of writing a `Dockerfile` from scratch, building applications in containers, building and pushing container images to remote repositories and more.

# [Overview](https://docs.docker.com/get-started/overview/)

Docker is an open platform for developing, shipping, and running applications. Docker enables you to separate your applications from your infrastructure so you can deliver software quickly. With Docker, you can manage your infrastructure in the same ways you manage your applications. By taking advantage of Docker’s methodologies for shipping, testing, and deploying code quickly, you can significantly reduce the delay between writing code and running it in production. Below, you will find a diagram representing the [architecture](https://docs.docker.com/get-started/overview/#docker-architecture).
![Architecture](https://docs.docker.com/assets/images/architecture.svg)

# [Get started](https://docs.docker.com/get-started/)

## Container registries

A container registry is a repository—or collection of repositories—used to store and access container images. Container registries can support container-based application development, often as part of DevOps processes. Container registries can connect directly to container orchestration platforms like Docker and Kubernetes. 

## [Dockerfile](https://docs.docker.com/engine/reference/builder/)
Docker can build images automatically by reading the instructions from a Dockerfile. A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image. This page describes the commands you can use in a Dockerfile. For example, lets say you want to build a `NodeJS` webapp using docker.

*Let's say that the webapp has the following directory as structure:*

```text
├── webapp/
│ ├── package.json
│ ├── README.md
│ ├── spec/
│ ├── src/
│   └── index.js
│ └── yarn.lock
│ └── Dockerfile
```

*As you can see in the example above, the directory `webapp/` contains a file called `Dockerfile`. Below, you will find the content of this file:*

```docker
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

*Now, we can `build` the application using the combination of the project source code along with the `Dockerfile` located at the root of the project repository.*

```bash
$ cd webapp/
$ docker build -t webapp:1.0.0 .
```

*Finally, we can push the final image to into our `container registry`.*

```bash
# eg: docker tag webapp:1.0.0 myRegistry.com/webapp:1.0.0
$ docker tag webapp:1.0.0 [REGISTRYHOST/][USERNAME/]NAME[:TAG]
$ docker push NAME[:TAG]
```

## Dockerfiles templates

TODO..

<font size=3>Angular application</font>

```docker
FROM node:latest AS builder
RUN mkdir /app
WORKDIR /app
COPY package*.json ./
RUN npm set progress=false && npm ci --force
COPY . .
RUN npm run ng -- build -c=development

FROM nginx:latest AS final
COPY --from=builder /app/dist/output /usr/share/nginx/html
CMD ["nginx", "-g", "daemon off;"]
EXPOSE 80
```

<font size=3>Java application</font>

```docker
FROM maven:3.9.3 AS builder
RUN mkdir /app /output
WORKDIR /app
COPY . .
RUN mvn package -Dmaven.test.failure.ignore=true
RUN mv /app/target/*.jar /output/app.jar

FROM openjdk:17.0.2-oracle AS final
COPY --from=builder /output/app.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
EXPOSE 9011
```

