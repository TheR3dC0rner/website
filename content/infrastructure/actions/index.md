---
title: "Building With Actions"
summary:  "Using Gitea Actions To Build Our AdaptixC2 Images"
date: 2025-09-20
tags: ["deployment", "actions", "scripting"]
draft: false
---

In the last article we got the development backend infrastructure up and running with Kubernetes and all its pieces.  We are going to start setting up our deployable command and control. 

To start with we are going to use AdaptixC2 for our deployment.  The first thing we want to do is create a custom docker image which we can use.  AdaptixC2 already provides a solid starting point to building an image and we are going to use it as a template.  

## Creating a docker service account

First, authenticate as the admin of our Gitea instance.  This would be the account one used when originally setting up Gitea.

 ![Pasted Image](<images/Pasted image 20250912141525.png>)
You then want to go to site administration which is on the right hand side under the profile picture.

![Pasted Image](<images/Pasted image 20250912141734.png>)
You then want to click on "Identity & Access" and "User Accounts"


![Pasted Image](<images/Pasted image 20250912141844.png>)

We are then going to create a dockersvc account

![Pasted Image](<images/Pasted image 20250912142241.png>)

This account will be used to push images to our organization.  

While we are also logged in as an admin we are going to create a token for a Gitea runner.

First select "Actions" and then Runners:

![Pasted Image](<images/Pasted image 20250912215054.png>)


Then select "Create new Runner"

![Pasted Image](<images/Pasted image 20250912214208.png>)

Store the generated token somewhere secure.

We are now going to want to authenticate with this account.
![Pasted Image](<images/Pasted image 20250912142351.png>)

We are going to want to generate an authentication token for this account.

Click on the account settings under the user profile.

![Pasted Image](<images/Pasted image 20250912142548.png>)
Under "Applications" we can generate a new token with limited privileges.  Once generated put the token in a secure location so we can use it later.

![Pasted Image](<images/Pasted image 20250912142939.png>)

Sign out once again and log into an account with your organization privileges.

Under your organization go to teams and create a new team for docker push.

![Pasted Image](<images/Pasted image 20250912143231.png>)

Add the team name

![Pasted Image](<images/Pasted image 20250912143332.png>)

Then modify its permissions.  These should also be very limited
![Pasted Image](<images/Pasted image 20250912143437.png>)

![Pasted Image](<images/Pasted image 20250912143533.png>)

Now click on that team and add our service account to it

![Pasted Image](<images/Pasted image 20250912143639.png>)

Now we can test everything
## Creating our docker image

To create our docker file we want to make a new repo.  In our Gitea instance we are going to go to our organization and create a new repository.

![Pasted Image](<images/Pasted image 20250912143738.png>)


Then on our jumpbox we are going to create a new directory for this repository.
```shell
mkdir docker-adaptixc2
```


We will now create a dockerfile:

dockerfile:

```yaml
FROM golang:1.24.4-bookworm AS builder
RUN git clone https://github.com/Adaptix-Framework/AdaptixC2 /app
RUN apt-get update && apt-get upgrade -y && apt-get install -y mingw-w64 make libssl-dev qt6-base-dev qt6-websockets-dev sudo libcap2-bin build-essential checkinstall zlib1g-dev libssl-dev qt6-declarative-dev qt6-scxml-dev
# client requires cmake version 3.28+
RUN wget https://github.com/Kitware/CMake/releases/download/v3.31.5/cmake-3.31.5.tar.gz && tar -zxvf cmake-3.31.5.tar.gz && cd cmake-3.31.5 && bash ./bootstrap && make && make install

WORKDIR /app

#COPY . .

RUN make server
RUN make extenders
RUN make client


FROM debian:bookworm-slim
RUN apt-get update --fix-missing \
    && apt-get -y upgrade \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && groupadd -g 999 adaptix \
    && useradd -r -u 999 -g adaptix adaptix \
    && mkdir -p /home/adaptix/ \
    && chown -R adaptix:adaptix /home/adaptix
WORKDIR /home/adaptix
COPY --from=builder /app/dist .

RUN chown -R adaptix:adaptix * \
    && chmod +x adaptixserver

USER adaptix
ENTRYPOINT ["./adaptixserver"]
EXPOSE 4321
EXPOSE 80
EXPOSE 443
EXPOSE 1080
```

This is based on the AdaptixC2 dockerfile in their repo but modified so we can build a pipeline around it.  The major changes are we clone the repo into the build section and we make a debian-slim image so we can later review logs from a command shell if needed.  We also set up user contexts so it will only run as a restricted user.  

The build can be smoke tested using podman which is installed on our jumpbox.

We can run the command
```shell
podman login
```
 Use dockersvc as the username, and the token as the password.
```shell
podman build  -t Gitea.dev.th3redc0rner.com/redcorner/adaptixc2:test -t adaptixc2:test .

podman run --rm adaptixc2:test
```

The output should be the adaptixserver help.  Now we can test our account push the image:

```shell
podman push Gitea.dev.th3redc0rner.com/redcorner/adaptixc2:test
```
If all goes well you should end up with a package in your Gitea repo.

![Pasted Image](<images/Pasted image 20250912162328.png>)
Once we know that it works and we can push the image we can register commit to our repo.

```shell
git init
git checkout -b main
git add.
git commit -m "first commit"
git remote add origin git@gitssh.dev.th3redc0rner.com:redCorner/docker-adaptixc2.git
git push -u origin main
```


## Setting up the build machine
Next, we are going to automate this build so it happens on the 15th of each month.  That way we can deploy with newer images without having to build it each time.  Gitea has actions built in, very similar to Github actions.  So what we build here should also work on a Github actions runner.  

To set up a runner we are going to create a new VM.  We are going to install docker on it using our ansible playbook. 

```shell
ansible-playbook -i gitaction.lan, docker-install.yaml
```

Once docker is installed we can ssh over to that machine.

Now create a directory for gitaction
```shell
mkdir /opt/gitaction
```

In that we are going to create a new docker-compose.yaml:

```yaml
version: "3.8"
services:
  runner:
    image: docker.io/Gitea/act_runner:nightly
    environment:
      CONFIG_FILE: /config.yaml
      Gitea_INSTANCE_URL: "${INSTANCE_URL}"
      Gitea_RUNNER_REGISTRATION_TOKEN: "${REGISTRATION_TOKEN}"
      Gitea_RUNNER_NAME: "${RUNNER_NAME}"
#      Gitea_RUNNER_LABELS: "${RUNNER_LABELS}"
    volumes:
      - ./config.yaml:/config.yaml
      - ./data:/data
      - /var/run/docker.sock:/var/run/docker.sock
```

We are now going to create the environment file:

.env:

```yaml
INSTANCE_URL="https://Gitea.dev.th3redc0rner.com/"
REGISTRATION_TOKEN="OR75NBwBD###################"
RUNNER_NAME="globalrunner"
#RUNNER_LABELS
```

This registers the runner with our Gitea server and assigns the runner its name.  Use the generated token from earlier. 

Finally this is the default config file which you can generate or copy to config.yaml.

config.yaml:

```yaml
# Example configuration file, it's safe to copy this as the default config file without any modification.

# You don't have to copy this file to your instance,
# just run `./act_runner generate-config > config.yaml` to generate a config file.

log:
  # The level of logging, can be trace, debug, info, warn, error, fatal
  level: info

runner:
  # Where to store the registration result.
  file: .runner
  # Execute how many tasks concurrently at the same time.
  capacity: 1
  # Extra environment variables to run jobs.
  envs:
    A_TEST_ENV_NAME_1: a_test_env_value_1
    A_TEST_ENV_NAME_2: a_test_env_value_2
  # Extra environment variables to run jobs from a file.
  # It will be ignored if it's empty or the file doesn't exist.
  env_file: .env
  # The timeout for a job to be finished.
  # Please note that the Gitea instance also has a timeout (3h by default) for the job.
  # So the job could be stopped by the Gitea instance if it's timeout is shorter than this.
  timeout: 3h
  # The timeout for the runner to wait for running jobs to finish when shutting down.
  # Any running jobs that haven't finished after this timeout will be cancelled.
  shutdown_timeout: 0s
  # Whether skip verifying the TLS certificate of the Gitea instance.
  insecure: false
  # The timeout for fetching the job from the Gitea instance.
  fetch_timeout: 5s
  # The interval for fetching the job from the Gitea instance.
  fetch_interval: 2s
  # The labels of a runner are used to determine which jobs the runner can run, and how to run them.
  # Like: "macos-arm64:host" or "ubuntu-latest:docker://docker.Gitea.com/runner-images:ubuntu-latest"
  # Find more images provided by Gitea at https://Gitea.com/docker.Gitea.com/runner-images .
  # If it's empty when registering, it will ask for inputting labels.
  # If it's empty when execute `daemon`, will use labels in `.runner` file.
  labels:
    - "ubuntu-latest:docker://docker.Gitea.com/runner-images:ubuntu-latest"
    - "ubuntu-22.04:docker://docker.Gitea.com/runner-images:ubuntu-22.04"
    - "ubuntu-20.04:docker://docker.Gitea.com/runner-images:ubuntu-20.04"

cache:
  # Enable cache server to use actions/cache.
  enabled: true
  # The directory to store the cache data.
  # If it's empty, the cache data will be stored in $HOME/.cache/actcache.
  dir: ""
  # The host of the cache server.
  # It's not for the address to listen, but the address to connect from job containers.
  # So 0.0.0.0 is a bad choice, leave it empty to detect automatically.
  host: ""
  # The port of the cache server.
  # 0 means to use a random available port.
  port: 0
  # The external cache server URL. Valid only when enable is true.
  # If it's specified, act_runner will use this URL as the ACTIONS_CACHE_URL rather than start a server by itself.
  # The URL should generally end with "/".
  external_server: ""

container:
  # Specifies the network to which the container will connect.
  # Could be host, bridge or the name of a custom network.
  # If it's empty, act_runner will create a network automatically.
  network: ""
  # Whether to use privileged mode or not when launching task containers (privileged mode is required for Docker-in-Docker).
  privileged: false
  # And other options to be used when the container is started (eg, --add-host=my.Gitea.url:host-gateway).
  options:
  # The parent directory of a job's working directory.
  # NOTE: There is no need to add the first '/' of the path as act_runner will add it automatically.
  # If the path starts with '/', the '/' will be trimmed.
  # For example, if the parent directory is /path/to/my/dir, workdir_parent should be path/to/my/dir
  # If it's empty, /workspace will be used.
  workdir_parent:
  # Volumes (including bind mounts) can be mounted to containers. Glob syntax is supported, see https://github.com/gobwas/glob
  # You can specify multiple volumes. If the sequence is empty, no volumes can be mounted.
  # For example, if you only allow containers to mount the `data` volume and all the json files in `/src`, you should change the config to:
  # valid_volumes:
  #   - data
  #   - /src/*.json
  # If you want to allow any volume, please use the following configuration:
  # valid_volumes:
  #   - '**'
  valid_volumes: []
  # overrides the docker client host with the specified one.
  # If it's empty, act_runner will find an available docker host automatically.
  # If it's "-", act_runner will find an available docker host automatically, but the docker host won't be mounted to the job containers and service containers.
  # If it's not empty or "-", the specified docker host will be used. An error will be returned if it doesn't work.
  docker_host: ""
  # Pull docker image(s) even if already present
  force_pull: true
  # Rebuild docker image(s) even if already present
  force_rebuild: false

host:
  # The parent directory of a job's working directory.
  # If it's empty, $HOME/.cache/act/ will be used.
  workdir_parent:
```

If we go to our admin login and check the runners you should see the registered runner actions.

![Pasted Image](<images/Pasted image 20250912214751.png>)

## Building automation

Now going back to our to new git repo we can create a workflow.

In the repo we are going create a directory called .gitea/workflows

```shell
mkdir -p .Gitea/workflows
```

We are then going to create a file called docker-build.yaml

The first thing we are going to do is make sure our global runner is working so lets build just a very simple test workflow.

docker-build.yaml 
```yaml
run-name: Adaptixc2_build
on: [push]


jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Clone the repository
        uses: actions/checkout@v4
      - name: date test
        run: echo "CURRENT_DATE=$(date +'%Y-%m-%d')" >> $GITHUB_ENV
      - name: Use the date variable
        run: echo "Todays date is ${{ env.CURRENT_DATE }}"

```

If this works we should get output similar to this:

![Pasted Image](<images/Pasted image 20250912215953.png>)

Now to build the proper workflow we have to give it our docker service username and password in our repo click.

To do this click Settings and then Actions:

![Pasted Image](<images/Pasted image 20250912220213.png>)


Under secrets we are going to create our password:

Click add Secret:

![Pasted Image](<images/Pasted image 20250912220328.png>)


The value should be the generated token from earlier.

Then under User Variables we want to create a new variable:
![Pasted Image](<images/Pasted image 20250912220444.png>)



Now that we know everything is working we can turn this into our full workflow. Modify the docker-build.yaml to look similar to this:

docker-build.yaml

```yaml
name: Adaptix_Build
run-name: Adaptix_Build
on:
  push:
  schedule:
    # Runs on the 15th day of every month at 02:00
    - cron: '0 2 15 * *'


jobs:
  adaptix_build:
    runs-on: ubuntu-latest
    steps:
      - name: Clone the repository
        uses: actions/checkout@v4
      - name: generate date
        run: echo "CURRENT_DATE=$(date +'%Y-%m-%d')" >> $GITHUB_ENV
      - name: docker login
        uses: docker/login-action@v3
        with:
          registry: Gitea.dev.th3redc0rner.com
          username: ${{ vars.DOCKERUSER }}
          password: ${{ secrets.DOCKERSVC_PASSWORD }}
      - name: Build and Push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            Gitea.dev.th3redc0rner.com/redcorner/adaptixc2:latest
            Gitea.dev.th3redc0rner.com/redcorner/adaptixc2:${{ env.CURRENT_DATE }}
```

This should do what we did manually earlier but give the image when pushed the tags latest and the current date.  

It will perform this action when we change something in the repo and scheduled at 2 am on the 15th of every month.

Don't forget to change your organization to your organization in the example files.


Your tree of the directory of the repo should look like this:

```
.
├── .Gitea
│   └── workflows
│       └── docker-build.yaml
└── Dockerfile
```


If you look under your organization and packages you should see the resulting docker images:
![Pasted Image](<images/Pasted image 20250912221354.png>)

![Pasted Image](<images/Pasted image 20250912221440.png>)


Now that we can build the image we can building our Kubernetes deployment automation.  In the next blog I will introduce my method of building helm charts by building a set of Kubernetes deployment files.
