# drone-example

This is a Drone CI/CD example for single machine.
I build up this example to show a simple CI/CD use case: build docker image, test, and then deploy the system.
I assume that you have the drone server and agent on the same machine, and you also want to deploy your system to that machine.
This setup helps us build up an example easier because we don't need to handle many secrets.
Don't use this example directly if someone you don't trust can push or send pull request to your repository.

## Preparation

- We will use GitHub's OAuth to login and get the repository, so you should register a new OAuth application [here](https://github.com/settings/applications/new).

- **Authorization callback URL** should be the URL you will run your drone server with `/authorize` suffix. For example, if you want to run your server on `http://123.123.123.123:8000`, then you must put `http://123.123.123.123:8000/authorize` here. Note that this must be a public IP, or GitHub cannot trigger the webhook.

- You can fill in whatever you want in other fields.

- Click **Register application** and then you will get your **Client ID** and **Client Secret**. You will need them later.

- Install the latest version of [Docker](https://docs.docker.com/install/) and [Docker Compose](https://docs.docker.com/compose/install/).

## Clone this repository

```bash
git clone https://github.com/ianlini/drone-example.git
cd drone-example
```

- You may want to fork this repository and clone your own repository so that you can push your commit.

## Setup Drone server

Add `.env` to `drone-server/` with the following content:

```bash
DRONE_ADMIN=...
DRONE_HOST=...
DRONE_SECRET=...
DRONE_GITHUB_CLIENT=...
DRONE_GITHUB_SECRET=...
```

- `DRONE_ADMIN`: your GitHub username
- `DRONE_HOST`: your host address (e.g., http://123.123.123.123:8000)
- `DRONE_SECRET`: generate a random string and put it here
- `DRONE_GITHUB_CLIENT`: the **Client ID** you got in the previous step
- `DRONE_GITHUB_SECRET`: the **Client Secret** you got in the previous step

You can start your server with the docker-compose command now:

```bash
cd drone-server
docker-compose up -d
```

Now you can access your Drone web server from http://123.123.123.123:8000, and login using GitHub's OAuth.

## Configure for your repository

- You should be able to understand the following steps on the GUI easily, but I still provide the URL in case you cannot find them.

- Turn on your repository in the repositories page (e.g., http://123.123.123.123:8000/account/repos).

- Set the repository to be trusted because we want to link the volume in `.drone.yml`.
  - If your repository is `ianlini/drone-example`, then the setting page may be something like http://123.123.123.123:8000/ianlini/drone-example/settings.
  - This is a dangerous setting, make sure you know what this is if you are not just testing.
  - If your repository is public, you can also disable **pull request** in **Repository Hooks** so that no one can trigger your webhook using pull request.

## Trigger the webhook

- Now you can try to trigger the webhook by pushing.

- After pushing, you will see the build here: http://123.123.123.123:8000/ianlini/drone-example/.

- You can also see that the image is built on your machine (`build-docker` step in the pipeline), and the flake8 (python linter) is run using that image (`flake8` step in the pipeline).

- Note that only `build-docker` and `flake8` is run here because we set `when` for other steps in the pipeline so that they won't be triggered when pushing.

## Deploy

- The deployment can only be triggered using the command-line tool.

- Install the tool following the instruction [here](http://docs.drone.io/cli-installation/).

- Find your token here: http://123.123.123.123:8000/account/token. This page will tell you how to use the token:

```bash
export DRONE_SERVER=...
export DRONE_TOKEN=...
drone info
```

- Now you can deploy a previous build using the command:

```bash
drone deploy {your repository} {build number} {environment}
```

- Example:

```bash
drone deploy ianlini/drone-example 3 production
```

- This will trigger the `deploy-production` step in the pipeline.

- Run `docker ps` and you will see that your container `droneexample_drone-example_1` is running.

- Run `docker logs droneexample_drone-example_1 -f` and you will see that it is printing the following things periodically:

```text
Hello world!!
Your secret is ''.
```

- The secret is empty here because you haven't set your secret yet.

## Configure the secret

- We pass a secret to the python file `hello_world.py` to show the power of the secret management in Drone. You should set it before deployment.

- Go to http://123.123.123.123:8000/ianlini/drone-example/settings/secrets and set a secret:
  - Secret Name: `example_secret`
  - Secret Value: `this is a secret`

- Now try to deploy again.

- Run `docker logs droneexample_drone-example_1 -f` and you will see that it is printing your `example_secret` periodically.

## Run locally

- Trigger `build-docker` and `flake8`:

```bash
drone exec --build-event push --commit-sha latest
```

- Trigger `deploy-local`:

```bash
EXAMPLE_SECRET='this is a local secret' drone exec --build-event deployment --build-target local --commit-sha latest
```

