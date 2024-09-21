This is a [Next.js](https://nextjs.org) project bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app).

## Getting Started

Add a `Dockerfile` to your root repository with exactly this name. In this file, we will add instructions for the docker image we will be building.

### Single-stage DockerFile

```bash
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD npm run dev
```

#### AND THAT’S IT for the Dockerfile!!

To build the docker Image from the Dockerfile we just created, simply run the following command. I name it `my-app`, but feel free to change it to what ever you want. Also don’t forget the `.` at last.

```basb
docker build -t my-app .
```

Once the image is created, we can run it by

```bash
docker run -p 3000:3000 my-app
```

`3000:3000` specify the the port you want to run the app on, I will run mine on 3000. And if you access `http://localhost:3000` , you will see your working app!

### Multi-stage DockerFile

```bash
FROM node:18-alpine as base
RUN apk add --no-cache g++ make py3-pip libc6-compat
WORKDIR /app
COPY package*.json ./
EXPOSE 3000

FROM base as builder
WORKDIR /app
COPY . .
RUN npm run build


FROM base as production
WORKDIR /app

ENV NODE_ENV=production
RUN npm ci

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs


COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/public ./public

CMD npm start

FROM base as dev
ENV NODE_ENV=development
RUN npm install 
COPY . .
CMD npm run dev
```

### Docker Compose

With Docker Compose, we don’t need to remember all those long commands to build or run containers. We can simply just use `docker-compose build` and `docker-compose up` .

Add a `docker-compose.yml` file to your root directory with the following conent.

```bash
version: '3.8'
services:
  app:
    image: openai-demo-app
    build:
      context: ./
      target: dev
      dockerfile: Dockerfile
    volumes:
        - .:/app
        - /app/node_modules
        - /app/.next
    ports:
      - "3000:3000"
```

### Testing with Docker Compose

We finally made our way to building the Docker image. We are going to use Docker’s BuildKit feature for a faster build.

```bash
COMPOSE_DOCKER_CLI_BUILD=1 DOCKER_BUILDKIT=1 docker-compose build
```

After the build finish, we can run the image and start the app by

```bash
docker-compose up
```

After that, if you go to `http://localhost:3000` on your browser, you will see your app running!!!

### Problems I encountered

- `Python is not set from command line or npm configuration`: (As a mac user) I tried to add python to path, alias python to python3, uninstall and reinstall python, etc.etc.etc. *NONE OF THOSE* worked out for me. So here are the two solutions I found. (1) use `node:18`(or any other number you prefer) instead of `node:18-alpine`. (2) add `RUN apk add — no-cache g++ make py3-pip libc6-compat` before you do any of the installations of your packages.

- `Could not find a production build in the ‘/app/.next’ directory` when using `docker-compose up`: Everything worked as expected if I ran the image through `docker run`. There are solutions out there suggesting add `CMD ["npm","run","build"]` right before `CMD npm start` in the Dockerfile. However, this gave me an error saying no two CMD allowed. My solution was to add `— /app/.next` to `volumes` in `Docker-compose.yml`.

[Source: ](https://readmedium.com/en/dockerize-a-next-js-app-4b03021e084d)