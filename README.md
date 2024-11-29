# Unit 7 Docker exercise

In this exercise, you will containerise a To-Do app using Docker. The To-Do app is in this repo, and is largely similar to Python applications you've seen before, especially if you've seen Poetry to manage Python dependencies in an earlier module. You'll create separate Docker images to develop, test and deploy a production-ready version of the app. We'll learn about writing custom Dockerfiles, multi-stage docker builds, configuration management and build optimisation.

## Setup: Install Docker

If you haven't already, you'll need to install [Docker Desktop](https://www.docker.com/products/docker-desktop). Installation instructions for Windows can be found [here](https://docs.docker.com/docker-for-windows/install/). If prompted to choose between using Linux or Windows containers during setup, make sure you choose Linux containers. If you can't install Docker locally, use an ACG VM.

# Exercise

## Part 1: Create a production container image

The primary goal of this exercise is to produce a Docker image that can be used to create containers that run the To-Do app in a production environment. 

Create a new file (called `Dockerfile`) in the root of the code repository. We'll include all the necessary Docker configuration in here. As a reminder, you can read more about dockerfile syntax [here](https://docs.docker.com/engine/reference/builder/).

### Create a minimal Dockerfile

The first step in creating a docker image is choosing a base image. We'll pick one from [Docker Hub](https://hub.docker.com/). A careful choice of base image can save you a lot of difficulty later, by providing many of your dependencies out-of-the-box. In this case, select one of the [official Python images](https://hub.docker.com/_/python). Available tags combine different operating systems (e.g. `buster`, `alpine`) with different Python versions. Select one that meets your Python version requirements. The operating system is less important: `buster` or `slim-buster` (Debian 10) will be fine, and most familiar.

When complete, you should have a single line in your Dockerfile:

```Dockerfile
FROM <base_image_tag>
```

You can build and run your Docker image with the following commands, although it won't do anything yet!

```bash
$ docker build --tag todo-app .
$ docker run todo-app
```

### Basic application installation

Expand the Dockerfile to include steps to import your application and launch it (this will look quite similar to the steps in your `README.md` file). 

You'll need to:

1. Install poetry
2. Copy across your application code
3. Install Python dependencies
4. Define an entrypoint / default launch command

Keep in mind a couple Docker best practices:

- Perform the "least changing" steps early, to fully take advantage of Docker's layer caching.
- Use `COPY` to move files into your image. Don't copy unnecessary files.
- Use `RUN` to execute shell commands as part of the build process.
- `ENTRYPOINT` and/or `CMD` define how your container will launch.
- Add an `EXPOSE` instruction to document which port your application is listening on.

When running `flask run` in your entrypoint, you'll need to tell Flask to accept requests coming in from outside the container. To do this, pass the `--host=0.0.0.0` option.
* Without this option, flask will only accept requests from `localhost` **inside the container**, which won't work here as we'll be making requests via the browser, which is running outside the container!

> If you want to run your app with [gunicorn](https://gunicorn.org/) (a true production-ready server), rather than using Flask's built in development server, you'll need to set an entrypoint like the following:
> ```
> poetry run gunicorn --bind 0.0.0.0 "todo_app.app:create_app()"
> ```

After updating your Dockerfile, rebuild your image and rerun it. You'll need two options for your `docker run` command:
* Load your .env file using the [`--env-file` option](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file).
* Publish the relevant port using the `--publish` option. This lets you connect to the container by visiting localhost on your computer (i.e. **outside the container**)

*Now view the app in your browser, and check it works!*

> <details> <summary style="font-size: 1.1rem">Running a Container in the Background</summary>
> <div markdown="1">By default, Docker attaches your current terminal to the container. The container will stop when you disconnect. If you want to launch your container in the background, use `docker run -d` to detach from the container, however while you're still changing things it's arguably easier to see what's going on without this. You can still view container logs using the `docker logs` command if you know the container's name or ID (if not, use `docker ps` to find the container first).
> 
> <div markdown="1"> If you want to stop a container that's running, you can first identify its name with `docker ps` and then use `docker stop <name>`.

> <details> <summary style="font-size: 1.1rem">Debugging a container</summary>
> It is possible to launch an interactive terminal (similar to an SSH session) when launching a container. The easiest way to do this is to run:
> <div markdown="1"> 
```
docker -it --entrypoint bash run
```
> In order for this to work your base image needs to have bash installed and shouldn't have a CMD statement in your Dockerfile

> <details> <summary style="font-size: 1.1rem">Running/Installing Poetry</summary>
> <div markdown="1">  When installing poetry [via the method described in the README.md](https://python-poetry.org/docs/#installing-with-the-official-installer) you may find that it is unavailable on the `PATH`
> <div markdown="1"> In this case you can either:
> * Add poetry to the `PATH` with an `ENV` command (you may want to try debugging the container to locate where poetry is installed)
> * Run `pip install poetry` instead which will install poetry to location available on the `PATH` (although please note [this install method technically isn't recommended by `poetry`](https://python-poetry.org/docs/#installing-manually) but will be fine as we are using a `.venv` folder)

More tips for getting your first Docker container working can be found [in the hints section below](#hints).

### Environment variables

There is a potential security issue when copying files into the image. The `.env` file contains application secrets (API keys), and it is included in the Docker image. This is bad practice. Anyone with access to the image (which you may make public) can discover the embedded content. 

It's good practice for containerised applications to be configured _only_ via environment variables, as they are a standard, cross-platform solution to configuration management. Instead of copying in a configuration file (`.env`) at build-time, we pass Docker the relevant environment variables at runtime (e.g. with `--env-file`). This will keep your secrets safe, while also keeping your image re-usable - you could spin up multiple containers, each using different credentials. Other settings that are not sensitive can also be varied between environments in this way. 

Create a `.dockerignore` file, and use to it specify files and directories that should never be copied to Docker images. This can include things like secrets ( `.env`) and any other unwanted files/directories (e.g. `.git`, `.vscode`, `.venv` or your Ansible files from the previous module).
Anything that will never be required to run or test your application should be registered with `.dockerignore` to improve your build speed and reduce the size of the resulting images. You can even ignore the Dockerfile itself. 

Even if you are being specific with your `COPY` commands, create the .dockerignore file anyway, because it's important ensure no one accidentally copies the .env file over in the future.

Note that any environment variables loaded as part of `docker run` will override any defined within the Dockerfile using the `ENV`.

## Part 2: Create a local development container

Containers are not only useful for production deployment. They can encapsulate the programming languages, libraries and runtimes needed to develop a project, and keep those dependencies separate from the rest of your system.

In Part 1 you created what's known as a "single-stage" docker image. It starts from a base image, adds some new layers and produces a new image that you can run. The resulting image can run the app in a production manner, but is not ideal for local development. Your local development image should have two key behaviours:

1. Enable Flask's debugging/developer mode to provide detailed logging and feedback.
2. Allow rapid changes to code files without having to rebuild the image each time.

To do this, you will convert your Dockerfile into a "multi-stage" docker file. Multi-stage builds can be used to generate different variants of a container (e.g. a development container, a testing container and a production container) from the same Dockerfile. You can read more about the technique [here](https://docs.docker.com/develop/develop-images/multistage-build/).


Here is an outline for a multi-stage build:

```dockerfile
FROM <base-python-image> as base

# Perform common operations, dependency installation etc...

FROM base as production

# Configure for production

FROM base as development

# Configure for local development
```

Since Flask's [development server](https://flask.palletsprojects.com/en/1.1.x/server/) allows hot reloading of code changes while running as long as we set the `FLASK_DEBUG` environment variable, we'll use that. The only thing we need to consider in order to achieve the second requirement is how to make changes to the code files and have them appear within the container without having to rebuild the image each time we modify the code. The solution is to use a [bind mount](https://docs.docker.com/storage/bind-mounts/) when running the container to make the "todo_app" directory on your host machine available as a mounted directory within the container.

The goal is to be able to create either a development or production image from the same Dockerfile, using commands like:

```bash
$ docker build --target development --tag todo-app:dev .
$ docker build --target production --tag todo-app:prod .
```

On UNIX shells, you can test the local development setup using a command similar to:

```bash
$ docker run --env-file ./.env -p 5100:80 --mount "type=bind,source=$(pwd)/todo_app,target=/app/todo_app" todo-app:dev
```
{: .wrap-if-needed}

## Part 3: Optimise your Dockerfile

Have a look at your Dockerfile again. Is there any way you could optimise it? Docker optimisation has two main considerations: speed and size. Optimising image size is challenging with an interpreted language like Python, as the application requires a Python interpreter, complete with standard library. You can, however, ensure that you only import what needed to run the application rather than the entire git repository. You can view the size of your docker images using `docker image ls`.

For this part, we'll focus on build speed optimisation.

Docker caches every layer it creates, making subsequent re-builds extremely fast. But that only works if the layers don't change. For example, Docker should not need to re-install your project dependencies because you apply a small bug fix to your application code.

Docker must rebuild a layer if:
1. The command in the Dockerfile changes
2. Files referenced by a `COPY` or `ADD` command are changed.
3. Any previous layer in the image is rebuilt.

You should place largely unchanging steps towards the top of your Dockerfile (e.g. installing build tools), and apply the move frequently changing steps towards the end (e.g. copying application code to the container).

With this in mind, review your Dockerfile. Is there any way you could improve it?

## Hints

- By default, docker runs commands inside the container as `root`. This means you should not need to worry about file permissions, nor execute any commands with `sudo`.
- Please note that variables defined in the `.env` file should *not contain quotes*. Flask is fine with this but Docker will treat the quotes as part of the value. Similarly you shouldn't use any spaces in the variable definitions around the `=` sign. [See here for Docker's syntax rules for the .env file](https://docs.docker.com/compose/env-file/#syntax-rules).
- You can use the `RUN` instruction to your Dockerfile to execute arbitrary shell commands (for example, to install poetry). By default, `RUN` uses the Bourne shell (`sh`), which may lack some features you want. If that's the case, use the [_exec_ form](https://docs.docker.com/engine/reference/builder/#run) to specify a different shell.
- Don't try to modify your shell environment as part of `RUN`. The changes won't persist to layers beyond the current `RUN` instruction. Instead, use Docker's `ENV` instruction to set persistent environment variables at build time.
- The `COPY` instruction can load data from the docker build context into the docker image.
- If Docker can't find your app code, are you sure you've copied it to the image? You may need to modify the `PYTHONPATH` environment variable.
- Think about how to install your dependencies using poetry. In particular, the `--no-root` and `-no-dev` flags may be of interest.
- (Hyper-V Backend Only) Folders used for bind mounts must be first added to the whitelist in docker settings under `Resources => File Sharing`.
![File Sharing Screenshot](../assets/images/docker_file_sharing.png)
- If you want to run a container without an executable but still want to keep the container running you can run the command `tail -f /dev/null`.
- If you're hitting an issue not covered, as always it's worth checking [the FAQs page](https://faq.devops.corndel.com/) for this exercise to see if there are any additional hints.


## Part 4 (Stretch Goal): Use Docker Compose

Launching containers with long `docker run` commands can become tedious, and difficult to share with other developers. Here we'll introduce **docker compose**, a tool bundled with docker that can automate the launch, networking, and lifecycle management of, containers. You can read more about it in [the docs](https://docs.docker.com/compose/). The basic principle is that you write all the configuration necessary to launch your containers in a single YAML file (`docker-compose.yml`), and a single command (`docker compose up`) can then be used to launch them.

1. Create a `docker-compose.yml` that launches and persists your development container.
2. Use `docker compose up` to launch your new, containerised local development environment.

## Part 5 (Stretch Goal): Debug Code Running in a Container

What use is a portable development container if you're unable to debug anything when something goes wrong?

### Prerequisites:
* [VSCode Docker Extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)
* [VSCode Remote Development Extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)

Create a new `debug` build target and compose file that mimics the development setup but only starts the container and **does not run flask** (see the hints for a clue on how to do this).

Run the container using `docker compose` and use the Docker extension to attach to the running container. From here open the container's project folder in VS Code. You should now be able to debug the flask app as if it was running outside your container.

_Important Note: You will likely need to install the Python VS Code extension inside the container before you can start debugging._

## Part 6 (Stretch Goal): Run your tests in Docker

1. Add a third build stage that encapsulates a complete test environment. Use the image to run your unit, integration and end-to-end tests with `docker run`. If you're not sure where to start, follow the guide on the [FAQs site](https://faq.devops.corndel.com/Modules/Module_5/Project_Exercise/testing_in_docker.html) until you can run all of your tests in a Docker container.
2. Modify your test container to be persistent, and re-run tests whenever it detects a change to the source files. You will need to run it with a bind mount and change your entrypoint to use run a tool that watches for file changes, e.g. [pytest-watch](https://pypi.org/project/pytest-watch/). On Windows, you will need to use the `--poll` option of pytest-watch for it to work inside Docker.
3. Expand your `docker-compose.yml` file to include your persistent test container(s). `docker-compose up` should now launch your local development environment and persistent test runners.
