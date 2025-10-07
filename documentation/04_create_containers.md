
### Step 1: Install Docker

First, you need Docker installed on your computer. It’s free and works on Windows, Mac, and Linux.

* Go to [docker.com/get-started](https://www.docker.com/get-started)
* Download and install Docker Desktop for your OS
* Once installed, open a terminal (Command Prompt, PowerShell, or Terminal)

---

### Step 2: Run Your First Container

Let’s run a simple container that shows “Hello World” to make sure everything is working.

Type this command in your terminal:

```bash
docker run hello-world
```

What happens:

* Docker checks if the `hello-world` container image is on your computer.
* If not, it downloads it from Docker Hub (a container image repository).
* Then Docker runs the container, which prints a friendly message.

---

### Step 3: Run a Web Server Container

Let’s run something more practical—a web server.

Run this command:

```bash
docker run -d -p 8080:80 nginx
```

Here’s what it means:

* `docker run`: tells Docker to start a container.
* `-d`: runs the container **in the background** (detached mode).
* `-p 8080:80`: maps port 80 inside the container (default web server port) to port 8080 on your computer.
* `nginx`: the name of the container image (a popular lightweight web server).

---

### Step 4: Access the Web Server

* Open your web browser.
* Go to: `http://localhost:8080`
* You should see the default **NGINX welcome page**!

---

### Step 5: Check Running Containers

To see running containers, run:

```bash
docker ps
```

You’ll see the container ID, image name, ports, and status.

---

### Step 6: Stop the Container

To stop your web server container:

1. Get the container ID from `docker ps`.
2. Run:

```bash
docker stop <container_id>
```

---

### Quick Summary:

* Docker runs containers from images.
* Containers are isolated, lightweight environments for apps.
* You can start/stop/manage containers with simple commands.

---

Do you want me to guide you on creating your **own container image** next, or should we jump to how Kubernetes runs and manages containers?
