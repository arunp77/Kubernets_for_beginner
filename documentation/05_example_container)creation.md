# 🧱 Step-by-Step: Create Your Own Container Image and push it to remote repo

Let’s make a basic web app using Python (don’t worry if you're not a developer — it’s simple!).

---
## Create a container image

### 🪄 Step 1: Create a Simple Web App

Create a folder on your system (e.g., `myapp`) and inside it, create a file called `app.py`:

```python
# app.py

from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "Hello from my container!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

This is a super simple Flask web server that returns a message.

> **Note**: Flask is a lightweight Python web framework.

---

### 📦 Step 2: Create a Dockerfile

Now, in the same folder, create a file called `Dockerfile` (no extension). This tells Docker how to build your image.

```Dockerfile
# Use an official Python base image
FROM python:3.9-slim

# Set working directory in the container
WORKDIR /app

# Copy app files into the container
COPY app.py /app

# Install Flask
RUN pip install flask

# Expose the port the app runs on
EXPOSE 5000

# Define the command to run the app
CMD ["python", "app.py"]
```

---

### 🛠️ Step 3: Build the Docker Image

Open a terminal in the same folder, and run:

```bash
docker build -t my-python-app .
```

This means:

* `docker build`: Build a Docker image.
* `-t my-python-app`: Tag the image with the name `my-python-app`.
* `.`: Build using the current directory.

Docker will read the `Dockerfile`, install everything, and create your image.

---

### 🚀 Step 4: Run Your Container

Once the image is built, run it:

```bash
docker run -d -p 5000:5000 my-python-app
```

Then go to:

```
http://localhost:5000
```

You should see:
**“Hello from my container!”** 🎉

---

### 🧹 Step 5: Stop and Remove the Container (Optional)

List running containers:

```bash
docker ps
```

Stop it:

```bash
docker stop <container_id>
```

Remove the container (optional):

```bash
docker rm <container_id>
```

---

####  ✅ You’ve Just:

* Written a web app
* Built your own Docker image
* Ran it in a container


## 🛰️ Step 1: Push Your Image to Docker Hub

### ✅ Prerequisites

1. **You need a Docker Hub account**
   👉 Go to [https://hub.docker.com](https://hub.docker.com) and sign up (if you haven’t already).

2. **Docker must be installed and logged in** on your system
   Run this in your terminal:

   ```bash
   docker login
   ```

   Enter your Docker Hub **username** and **password** when prompted.

---

### 🪪 Step 2: Tag Your Image

Docker images must be named like this before pushing:

```
<dockerhub-username>/<image-name>:<tag>
```

Assuming your Docker Hub username is `yourusername`, and your image is `my-python-app`, run:

```bash
docker tag my-python-app yourusername/my-python-app:latest
```

> Replace `yourusername` with your actual Docker Hub username.

---

### 🚀 Step 2: Push the Image

Now push the image to Docker Hub:

```bash
docker push yourusername/my-python-app:latest
```

This will upload your image layers to your Docker Hub repository.

You should see output like:

```
The push refers to repository [docker.io/yourusername/my-python-app]
...
latest: digest: sha256:xxxxxxxxxxx size: xxxx
```

---

### 🔍 Step 3: Verify on Docker Hub

1. Go to: [https://hub.docker.com/repositories](https://hub.docker.com/repositories)
2. You should see `yourusername/my-python-app` listed there.
3. Click it to confirm your image is now in the cloud!

---

### 🎉 Success!

You’ve now:

* Created a custom Docker image
* Uploaded (pushed) it to Docker Hub

This means: **anyone in the world** (or any cloud server, or Kubernetes cluster) can now pull and run your app using:

```bash
docker run yourusername/my-python-app
```

