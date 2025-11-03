
# Lab: Jenkins on Docker (Windows / macOS / Linux)

## What you’ll achieve

* Run **Jenkins LTS** in Docker
* Persist data to a volume/bind-mount
* Unlock and configure Jenkins
* (Optional) Enable Jenkins to build Docker images
* Learn safe upgrade/backup steps + troubleshooting

---

## Prerequisites

* **Docker Desktop** (Windows/macOS) or **Docker Engine** (Linux) installed and running.
* Admin/sudo rights.
* Port **8080** available (or choose another).

Verify:

```bash
docker --version
docker info
```

---

## Option A — Quick Start with Docker CLI

### 1) Create a persistent volume (recommended)

```bash
docker volume create jenkins_home
```

> This keeps Jenkins config/jobs/plugins across container restarts/upgrades.

### 2) Run Jenkins LTS container

**Windows/macOS/Linux (basic CI server):**

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

* `8080` → Web UI
* `50000` → JNLP inbound agents (keep if you’ll use agents later)

> If port 8080 is busy, change `-p 9090:8080` and use `http://localhost:9090`.

### 3) Get initial admin password

Wait ~30–60s, then:

```bash
docker logs jenkins 2>&1 | findstr /i password   # Windows PowerShell
# or
docker logs jenkins 2>&1 | grep -i password      # macOS/Linux
```

You’ll see a line with the initial admin password (it also lives inside the container at `/var/jenkins_home/secrets/initialAdminPassword`).

### 4) Open Jenkins

Go to **[http://localhost:8080](http://localhost:8080)** (or your alternate port) → **Unlock Jenkins**
Paste the password → **Install suggested plugins** → create **admin** user → confirm Jenkins URL.

### 5) Smoke test job

* **New Item → Freestyle project → hello-docker-jenkins**
* **Build step**: Execute shell / Windows batch command:

  * Windows:

    ```bat
    echo Hello from Jenkins in Docker on Windows!
    ```
  * Linux/macOS:

    ```bash
    echo "Hello from Jenkins in Docker!"
    ```
* **Save → Build Now → Console Output = SUCCESS**

---

## Option B — Use Docker Compose (clean & repeatable)

### 1) Create project folder and Compose file

```bash
mkdir jenkins-docker && cd jenkins-docker
```

**docker-compose.yml**

```yaml
services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
volumes:
  jenkins_home:
```

### 2) Start/Stop

```bash
docker compose up -d
docker compose ps
docker compose logs -f
# Stop
docker compose down
```

Unlock/configure Jenkins as in Option A.

---

## (Optional) Enable Jenkins to build Docker images

You have two common patterns:

### Pattern 1 — “Docker outside of Docker” (mount Docker socket)

* **Linux/macOS**:

  ```yaml
  services:
    jenkins:
      image: jenkins/jenkins:lts
      ports:
        - "8080:8080"
      volumes:
        - jenkins_home:/var/jenkins_home
        - /var/run/docker.sock:/var/run/docker.sock
  volumes:
    jenkins_home:
  ```

  Install Docker CLI inside Jenkins container (once):

  ```bash
  docker exec -it jenkins bash
  # Inside container:
  curl -fsSL https://get.docker.com -o get-docker.sh
  sh get-docker.sh
  exit
  ```

  Now pipeline steps like `docker build` will talk to the host daemon via the socket.

* **Windows (Docker Desktop)**: socket is a **named pipe**. Easiest route is to **use an agent** on the host for Docker builds, or switch Docker Desktop to **WSL2 backend** and mount the Linux socket (`/var/run/docker.sock`) by running Jenkins on a Linux VM/WSL. (Socket mounting on Windows native containers is not supported.)

### Pattern 2 — Use a dedicated Docker agent

Run builds on an agent that already has Docker (or use Kubernetes agents). Keep the controller lightweight.

---

## Useful Management Commands

```bash
# See container status
docker ps

# Follow logs
docker logs -f jenkins

# Restart container
docker restart jenkins

# Exec into container shell
docker exec -it jenkins bash

# Backup the named volume to a tar (Linux/macOS):
docker run --rm -v jenkins_home:/data -v "$PWD":/backup alpine \
  sh -c "cd /data && tar czf /backup/jenkins_home_backup.tgz ."
# The file jenkins_home_backup.tgz will be in your current folder
```

---

## Upgrading Jenkins (safe approach)

1. **Backup** your volume (see above).
2. Pull the latest image:

   ```bash
   docker pull jenkins/jenkins:lts
   ```
3. Recreate container:

   * CLI:

     ```bash
     docker stop jenkins && docker rm jenkins
     docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
       -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
     ```
   * Compose:

     ```bash
     docker compose pull
     docker compose up -d
     ```
4. Verify plugins after upgrade (**Manage Jenkins → Plugin Manager**). Update plugins if needed.

---

## Bind-mount alternative (see data on host)

Instead of a named volume:

```bash
# Windows PowerShell (adjust path)
mkdir C:\jenkins_home
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 ^
  -v "C:\jenkins_home:/var/jenkins_home" jenkins/jenkins:lts
```

```bash
# macOS/Linux
mkdir -p $HOME/jenkins_home
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v $HOME/jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

---

## Troubleshooting

**Port already in use (8080)**

* Change mapping: `-p 9090:8080` and visit `http://localhost:9090`.

**Jenkins won’t start / keeps restarting**

* Check logs: `docker logs -f jenkins`
* Ensure disk space and that the volume/bind-mount is writable.

**Can’t install plugins (corporate proxy)**

* **Manage Jenkins → Plugins → Advanced** → set **HTTP/HTTPS proxy**.
* Or set env vars on container: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`.

**Permission issues on Linux bind-mount**

* Set ownership on host:

  ```bash
  sudo chown -R 1000:1000 $HOME/jenkins_home
  ```

  Jenkins runs as uid 1000 by default.

**Windows named pipe for Docker builds**

* Prefer a Linux agent/WSL2 or use the socket on a Linux host. Building Docker inside the Windows-hosted controller is tricky due to pipe mounts.

---

## Quick Validation Checklist

* [ ] `docker ps` shows **jenkins** container **Up**
* [ ] Open **[http://localhost:8080](http://localhost:8080)** → unlocked → admin created
* [ ] Installed suggested plugins
* [ ] Ran a **hello** freestyle job successfully
* [ ] (Optional) Built a Docker image from Jenkins using socket/agent

---

If you want, I can generate a **ready-to-run** folder with:

* `docker-compose.yml` (socket + volume variants)
* A sample pipeline `Jenkinsfile` that builds a Docker image
  Just say “prepare the starter pack” and I’ll drop it here.
