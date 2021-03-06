# Nodejs Prometheus
I gonna setup prometheus to monitor my nodejs app.

But first I need to contact the nodejs app and push docker image to container registry. 
Since I'm more familiar with GitHub Container Registry I'll push the image to my personal registry and I gonna use a GitHub action for that.

## 1. Containerize the app
In the root of the app folder I've create a Dockerfile and following instructions.
```bash
FROM node:alpine3.11

RUN mkdir /app

WORKDIR /app

COPY . . 

RUN npm install

EXPOSE 3000

CMD ["npm", "run", "start"]
```
also I have used `.dockerignore` to which I don't want to containerize with the app such as `node_modules` and `k8s` folders.

If I locally building the image I should use following commands to build the image and run the container.
#### Build Docker image
```shell
docker build -t {DOCKER_PROFILE_USERNAME}/node-app:latest .
```


#### Run a Docker container
```shell
docker run -p 3000:3000 chamodshehanka/node-app
```

## 2. Publish the image to GitHub Container Registry using a GitHub Action

To publish a Docker image to GitHub Container I need to have a personal access token. Follow [this](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) guide for more information.

Then go to repsitory settings and add that token as a secret (Secret name is `GHCR_TOKEN`). If you don't know about GitHub secrets please go through [this](https://docs.github.com/en/actions/security-guides/encrypted-secrets) guide.

Then add below file content to `.github/workflows/docker-publish.yml` file.
```yaml
name: GitHub Registry Image Push
on:
  push:
    branches:
      - main
      
jobs:
  build-push:
    runs-on:
      - ubuntu-latest
    env:
      SERVICE_NAME: ghcr.io/shehanka/node-app
      CONTAINER_REGISTRY: ghcr.io
    steps:
      - uses: actions/checkout@v2
      - name: Set Version
        id: event-version
        run: echo ::set-output name=SOURCE_TAG::${GITHUB_REF#refs/tags/}
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
            path: /tmp/.buildx-cache
            key: ${{ runner.os }}-buildx-${{ github.sha }}
            restore-keys: |
              ${{ runner.os }}-buildx-
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ${{ env.CONTAINER_REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_TOKEN }}
      - name: Build the Docker image
        run: docker build --tag ${SERVICE_NAME}:latest --file Dockerfile .
      - name: GitHub Image Push
        run: docker push $SERVICE_NAME:latest
```

Then I can see my image in the [GitHub Container Registry](https://github.com/Shehanka/nodejs-prometheus/pkgs/container/node-app).

and I can even pull that image to my local machine using below command.

```shell
docker pull ghcr.io/shehanka/node-app:latest
``` 

