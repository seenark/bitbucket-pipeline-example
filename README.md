# bitbucket-pipeline-example

this is my note that contain about how to do **Bitbucket pipeline** to build docker image to _dockerhub_ after that deploy to **Google VM Instance**

# Create Google VM

1. it very easy to do in [google cloud console](https://console.cloud.google.com/compute/instances?project=wedding-card-383908)
2. install [docker](https://docs.docker.com/engine/install/ubuntu/)

# Create user in VM

1. SSH into VM by using your user via browser console

2. create deployer user

```bash
sudo groupadd docker
getent group docker # check the group
awk -F':' '/docker/{print $4}' /etc/group # check who are in the group
sudo adduser deployer  # Replace 'deployer' with your desired username
sudo usermod -aG sudo deployer
sudo usermod -aG docker deployer

awk -F':' '/docker/{print $4}' /etc/group # check who are in the group
```

Restart docker

```bash
sudo systemctl restart docker.service
sudo systemctl status docker.service
```

# Create SSH key

1. create _private & public key_ in your local machine

```bash
ssh-keygen -t rsa -b 4096 -C "deployer@example.com"
```

2. turn the _private key into Base64 encoded_

```bash
cat <private key filename> | base64
```

store the _private key base64 encoded_ in **Bitbucket pipeline variables** in bitbucket website

3. copy _public key_ and paste into _VM config page_ via _google cloud console_

to see _public key in terminal_

```bash
cat <public key filename>
```

# Write the Bitbucket pipeline

create file `bitbucket-pipelines.yml` at your root project

```yaml
definitions:
  steps:
    - step: &hi
        name: Hi
        script:
          - echo Hi Bitbucket pipelines
# image: docker:stable
image: atlassian/default-image:4
pipelines:
  custom:
    dockerprune:
      - step:
          name: docker prune
          script:
            - eval $(ssh-agent -s)
            # - apt-get update && apt-get install -y openssh-client
            - echo $VM_IP
            - mkdir -p ~/.ssh
            - echo $(echo "$SSH_PRIVATE_KEY" | base64 -d) > ~/.ssh/ID_RSA
            # - cat ~/.ssh/ID_RSA | tr -d '\r' | ssh-add -
            - echo "$(echo "$SSH_PRIVATE_KEY" | base64 -d)" | tr -d '\r' | ssh-add -
            - touch ~/.ssh/config
            - touch ~/.ssh/known_hosts
            - chmod -R 400 ~/.ssh
            # copy docker-compose.dockerhub.yaml to VM
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && docker system prune -a -f"
    web:
      - step:
          name: build docker image
          script:
            # - npm i -g just-install
            - IMAGE_NAME=plai-web
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - curl -1sLf  'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.deb.sh' | bash
            - apt-get update && apt-get install -y infisical
            - INFISICAL_TOKEN=$INFISICAL_TOKEN infisical --domain=$INFISICAL_URL --env=staging run -- docker build -f apps/web/Dockerfile --platform linux/amd64 -t $IMAGE .
            # - just docker-web-build
            - docker save $IMAGE --output "${IMAGE_NAME}.tar"
          services:
            - docker
          caches:
            - docker
          artifacts:
            - "*.tar"
      # push docker image to registry
      - step:
          name: Public web image to Dockerhub
          script:
            - echo ${DOCKERHUB_PASSWORD} | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
            - IMAGE_NAME=plai-web
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - docker load --input "${IMAGE_NAME}.tar"
            - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - docker tag "${IMAGE}" "${IMAGE}:${VERSION}"
            - docker push "${IMAGE}:${VERSION}"
          services:
            - docker
      # deploy web
      - step:
          name: Deploy to Google Compute engine
          deployment: Staging
          script:
            - IMAGE_NAME=plai-web
            # - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - VERSION="staging-0.1.51"
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${VERSION}
            # setup this container that make it can communicate with VM via ssh
            - eval $(ssh-agent -s)
            # - apt-get update && apt-get install -y openssh-client
            - echo $VM_IP
            - mkdir -p ~/.ssh
            - echo $(echo "$SSH_PRIVATE_KEY" | base64 -d) > ~/.ssh/ID_RSA
            # - cat ~/.ssh/ID_RSA | tr -d '\r' | ssh-add -
            - echo "$(echo "$SSH_PRIVATE_KEY" | base64 -d)" | tr -d '\r' | ssh-add -
            - touch ~/.ssh/config
            - touch ~/.ssh/known_hosts
            - chmod -R 400 ~/.ssh
            # copy docker-compose.dockerhub.yaml to VM
            - scp -o BatchMode=yes -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA -r ./docker-compose.dockerhub-web.yaml deployer@$VM_IP:docker-compose.dockerhub-web.yaml
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "docker pull $IMAGE"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && INFISICAL_TOKEN=$INFISICAL_TOKEN FRONTEND_IMAGE=$IMAGE infisical --domain=$INFISICAL_URL --env=staging run -- docker compose -f docker-compose.dockerhub-web.yaml rm --stop --force --volumes web"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && INFISICAL_TOKEN=$INFISICAL_TOKEN FRONTEND_IMAGE=$IMAGE infisical --domain=$INFISICAL_URL --env=staging run -- docker compose -f docker-compose.dockerhub-web.yaml up web -d"
    api:
      # - step: *hi
      # build docker image
      - step:
          name: Build and Test
          script:
            - npm i -g just-install
            - IMAGE_NAME=plai-api
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - just docker-api-build-pipeline $IMAGE
            - docker save $IMAGE --output "${IMAGE_NAME}.tar"
          services:
            - docker
          caches:
            - docker
          artifacts:
            - "*.tar"
      # push docker image to registry
      - step:
          name: Public image to Dockerhub
          script:
            - echo ${DOCKERHUB_PASSWORD} | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
            - IMAGE_NAME=plai-api
            - docker load --input "${IMAGE_NAME}.tar"
            - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - docker tag "${IMAGE}" "${IMAGE}:${VERSION}"
            - docker push "${IMAGE}:${VERSION}"
          services:
            - docker
      # deploy api
      - step:
          name: Deploy to Google Compute engine
          deployment: Staging
          script:
            - IMAGE_NAME=plai-api
            - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${VERSION}
            # setup this container that make it can communicate with VM via ssh
            - eval $(ssh-agent -s)
            # - apt-get update && apt-get install -y openssh-client
            - echo $VM_IP
            - mkdir -p ~/.ssh
            - echo $(echo "$SSH_PRIVATE_KEY" | base64 -d) > ~/.ssh/ID_RSA
            # - cat ~/.ssh/ID_RSA | tr -d '\r' | ssh-add -
            - echo "$(echo "$SSH_PRIVATE_KEY" | base64 -d)" | tr -d '\r' | ssh-add -
            - touch ~/.ssh/config
            - touch ~/.ssh/known_hosts
            - chmod -R 400 ~/.ssh
            # copy docker-compose.dockerhub.yaml to VM
            - scp -o BatchMode=yes -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA -r ./docker-compose.dockerhub.yaml deployer@$VM_IP:docker-compose.dockerhub.yaml
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "docker pull $IMAGE"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && API_IMAGE=$IMAGE docker compose -f docker-compose.dockerhub.yaml rm --stop --force --volumes api"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && INFISICAL_TOKEN=$INFISICAL_TOKEN API_IMAGE=$IMAGE infisical --domain=$INFISICAL_URL --env=staging run -- docker compose -f docker-compose.dockerhub.yaml up -d api"
          services:
            - docker
  branches:
    staging0:
      # - step: *hi
      # build docker image
      - step:
          name: Build and Test
          script:
            - npm i -g just-install
            - IMAGE_NAME=plai-api
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - just docker-api-build-pipeline $IMAGE
            - docker save $IMAGE --output "${IMAGE_NAME}.tar"
          services:
            - docker
          caches:
            - docker
          artifacts:
            - "*.tar"
      # push docker image to registry
      - step:
          name: Public image to Dockerhub
          script:
            - echo ${DOCKERHUB_PASSWORD} | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
            - IMAGE_NAME=plai-api
            - docker load --input "${IMAGE_NAME}.tar"
            - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}
            - docker tag "${IMAGE}" "${IMAGE}:${VERSION}"
            - docker push "${IMAGE}:${VERSION}"
          services:
            - docker
      # deploy api
      - step:
          name: Deploy to Google Compute engine
          deployment: Staging
          script:
            - IMAGE_NAME=plai-api
            - VERSION="staging-0.1.${BITBUCKET_BUILD_NUMBER}"
            - IMAGE=${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${VERSION}
            # setup this container that make it can communicate with VM via ssh
            - eval $(ssh-agent -s)
            # - apt-get update && apt-get install -y openssh-client
            - echo $VM_IP
            - mkdir -p ~/.ssh
            - echo $(echo "$SSH_PRIVATE_KEY" | base64 -d) > ~/.ssh/ID_RSA
            # - cat ~/.ssh/ID_RSA | tr -d '\r' | ssh-add -
            - echo "$(echo "$SSH_PRIVATE_KEY" | base64 -d)" | tr -d '\r' | ssh-add -
            - touch ~/.ssh/config
            - touch ~/.ssh/known_hosts
            - chmod -R 400 ~/.ssh
            # copy docker-compose.dockerhub.yaml to VM
            - scp -o BatchMode=yes -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA -r ./docker-compose.dockerhub.yaml deployer@$VM_IP:docker-compose.dockerhub.yaml
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "docker pull $IMAGE"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && API_IMAGE=$IMAGE docker compose -f docker-compose.dockerhub.yaml rm --stop --force --volumes api"
            - ssh -o StrictHostKeyChecking=no -i ~/.ssh/ID_RSA deployer@$VM_IP "bash -c ls -l && INFISICAL_TOKEN=$INFISICAL_TOKEN API_IMAGE=$IMAGE infisical --domain=$INFISICAL_URL --env=staging run -- docker compose -f docker-compose.dockerhub.yaml up -d api"
          services:
            - docker
```
