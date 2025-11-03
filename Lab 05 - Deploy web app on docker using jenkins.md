
# Lab: Deploy a Static Website on Docker using Jenkins

## What you’ll build

* A tiny static site (`index.html`) served by **Nginx** in a Docker container.
* A **Jenkins Declarative Pipeline** that:

  1. checks out code
  2. builds a Docker image
  3. stops & removes any existing container
  4. runs the new container on a chosen port
  5. health-checks the deployment

---

## Prerequisites

* **Docker** running (`docker info` works).
* **Jenkins** running and reachable (e.g., `http://localhost:8080`).
* Jenkins must be able to run `docker` commands:

  * **Linux/macOS**: mount Docker socket into Jenkins container or run Jenkins on the host with Docker CLI available.
  * **Windows**: easiest is to run builds on an **agent** with Docker, or run Jenkins on WSL2/Linux VM with `/var/run/docker.sock` mounted.
* A Git repo (GitHub/GitLab/Bitbucket or local), accessible from Jenkins.

> If you want to push to Docker Hub or a registry, you can add that later (optional step included).

---

## Step 1 — Create Project Structure (locally)

```
static-site/
├─ Jenkinsfile
├─ Dockerfile
└─ site/
   └─ index.html
```

**site/index.html**

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Static Site on Docker via Jenkins</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    body { font-family: system-ui, Arial; margin: 3rem; }
    .card { border: 1px solid #ddd; padding: 1.5rem; border-radius: 12px; }
    h1 { margin-top: 0; }
    .ok { color: #0a0; font-weight: 700; }
  </style>
</head>
<body>
  <div class="card">
    <h1>🚀 Deployed by Jenkins</h1>
    <p class="ok">Hello! This static site is running inside Nginx in Docker.</p>
    <p>Change this file, push, and Jenkins will redeploy.</p>
  </div>
</body>
</html>
```

**Dockerfile**

```dockerfile
# Tiny Nginx image
FROM nginx:alpine

# Remove default page and add ours
RUN rm -rf /usr/share/nginx/html/*
COPY site/ /usr/share/nginx/html/

# Expose port from container
EXPOSE 80

# Default command (from base image)
```

Commit and push to your repository.

---

## Step 2 — Add the Jenkins Pipeline (Jenkinsfile)

> This pipeline builds the image, replaces any running container, and starts the new version.

**Jenkinsfile**

```groovy
pipeline {
  agent any

  environment {
    APP_NAME = "static-site"
    IMAGE_TAG = "${env.BUILD_NUMBER}"          // e.g., 15
    DOCKER_IMAGE = "local/${env.APP_NAME}:${env.IMAGE_TAG}"
    HOST_PORT = "8081"                          // change if you want a different port
    CONTAINER_NAME = "${env.APP_NAME}"
  }

  options { timestamps() }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          docker build -t ${DOCKER_IMAGE} .
        '''
      }
    }

    stage('Stop & Remove Old Container (if exists)') {
      steps {
        sh '''
          if [ "$(docker ps -aq -f name=${CONTAINER_NAME})" ]; then
            docker rm -f ${CONTAINER_NAME} || true
          fi
        '''
      }
    }

    stage('Run New Container') {
      steps {
        sh '''
          docker run -d --name ${CONTAINER_NAME} \
            -p ${HOST_PORT}:80 \
            ${DOCKER_IMAGE}
        '''
      }
    }

    stage('Health Check') {
      steps {
        script {
          // Try curl a few times to allow container to start
          def tries = 10
          def ok = false
          for (int i = 0; i < tries; i++) {
            def code = sh(returnStatus: true, script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${HOST_PORT}")
            if (code == 200) { ok = true; break }
            sleep 2
          }
          if (!ok) {
            error "Health check failed: site not responding on http://localhost:${HOST_PORT}"
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployed: http://localhost:${HOST_PORT}"
    }
    always {
      sh 'docker images | head -n 15 || true'
      sh 'docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"'
    }
  }
}
```

### Windows notes

If your Jenkins agent runs on **Windows** and you’re using a **Freestyle** job with Windows batch, you can switch steps to `bat` and commands accordingly. For **Pipeline**, the `sh` step requires a POSIX shell; easiest is to use a Linux agent/WSL2 containerized agent. (Alternative: use `bat` steps and equivalent PowerShell/CMD.)

---

## Step 3 — Configure Jenkins Credentials (if needed)

If your repository is private:

1. **Manage Jenkins → Credentials** → add Git credentials (username/password or SSH key).
2. In your **Pipeline job**, set **SCM** to your repo and select credentials (or put a `git` step in the `Jenkinsfile` with credentials).

---

## Step 4 — Create the Pipeline Job

1. In Jenkins: **New Item → Pipeline** → name: `static-site-docker`.
2. **Pipeline** tab → choose:

   * **Definition**: *Pipeline script from SCM*
   * **SCM**: *Git*
   * **Repository URL**: your repo
   * **Credentials**: (if required)
   * **Script Path**: `Jenkinsfile`
3. **Save**.

---

## Step 5 — First Run & Validate

1. Click **Build Now**.
2. Open the build → **Console Output**:

   * Should show Docker build logs.
   * Old container (if any) removed.
   * New container started on **host port 8081**.
3. Verify in the browser: **[http://localhost:8081](http://localhost:8081)**
   You should see the “Deployed by Jenkins” page.

**CLI checks**

```bash
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
# Should list: static-site  local/static-site:<build#>  0.0.0.0:8081->80/tcp
```

---

## Step 6 — Make a Change & Watch Redeploy

1. Edit `site/index.html` (change a line).
2. Commit and push.
3. Run the pipeline again (or set a webhook so SCM push auto-triggers).
4. Refresh **[http://localhost:8081](http://localhost:8081)** (hard refresh) to see the new version.

---

## Optional: Auto-Trigger on Push (GitHub)

* Install **GitHub** plugin (if needed).
* In GitHub repo → **Settings → Webhooks** → Add:

  * Payload URL: `http://<jenkins-host>/github-webhook/`
  * Content type: `application/json`
  * Events: Just the push event (or default).
* In Jenkins job → **Build Triggers** → “GitHub hook trigger for GITScm polling”.

---

## Optional: Push Image to Docker Hub (or Registry)

1. In Jenkins: **Credentials → Global** → add **Username/Password** for Docker Hub (ID: `dockerhub`).
2. Add a **post-build** stage to push:

   ```groovy
   stage('Push to Docker Hub') {
     when { expression { return env.BRANCH_NAME == 'main' } } // optional
     steps {
       sh '''
         echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
         docker tag ${DOCKER_IMAGE} ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}
         docker push ${DOCKERHUB_USER}/${APP_NAME}:${IMAGE_TAG}
       '''
     }
   }
   ```

   And expose creds via environment or with `withCredentials`.

---

## Optional: Use Docker Compose for Run Step

If you prefer Compose:

**docker-compose.yml (in repo)**

```yaml
services:
  web:
    image: local/static-site:${IMAGE_TAG:-latest}
    container_name: static-site
    ports:
      - "8081:80"
```

Change the “Run New Container” step:

```groovy
sh '''
  docker compose down || true
  IMAGE_TAG=${IMAGE_TAG} docker compose up -d
'''
```

---

## Clean-up Commands

```bash
# Stop & remove container
docker rm -f static-site

# Remove images matching local/static-site
docker images "local/static-site" --format "{{.Repository}}:{{.Tag}}" | xargs -r docker rmi
```

---

## Troubleshooting

**Port already in use**

* Change `HOST_PORT` in Jenkinsfile (e.g., 8082) and re-run.

**Jenkins can’t run docker**

* Make sure Jenkins agent has Docker CLI and permission to access Docker daemon

  * Linux: mount `/var/run/docker.sock` into Jenkins container, or run on a host agent with Docker.
  * Windows: prefer Linux agent/WSL2 for simplicity.

**Permission denied on bind-mounts**

* If you switch to bind-mounts, ensure correct ownership (Linux: `chown 1000:1000`).

**Health check fails**

* Inspect logs: `docker logs static-site`
* Verify container running: `docker ps`
* Test locally: `curl http://localhost:8081`

---

## Validation Checklist

* [ ] Pipeline builds Docker image successfully
* [ ] Old container is removed on each run
* [ ] New container is up and reachable on configured port
* [ ] Site returns **HTTP 200** (health check passes)
* [ ] Code change triggers redeploy

---

