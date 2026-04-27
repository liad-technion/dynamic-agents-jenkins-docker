# Task — Dynamic Docker Agents on Jenkins Main

**Goal:** configure the `main` Jenkins controller (defined in `docker-compose.yml`) to talk to the host Docker engine and use it as a cloud provider for **dynamic, ephemeral build agents** in a Declarative Pipeline.

**Reference compose file:** `99_misc/setup/docker/docker-compose.yml`

---

## Background

Looking at the `main` service in `docker-compose.yml`:

```yaml
main:
    hostname: jenkins-main
    build:
        context: .
        dockerfile: Dockerfile.main
    ports:
        - "80:8080"
    volumes:
        - /var/run/docker.sock:/var/run/docker.sock   # <-- host docker engine
        - ./jenkins_home:/var/jenkins_home
```

The host Docker socket is already bind-mounted into the controller, and `Dockerfile.main` installs `docker-ce-cli`. That means the Jenkins controller can already *see* Docker — your job is to wire it into Jenkins so pipelines can spin up containers as agents on demand.

By the end of this task, every `stage` you mark with a `docker` agent block will:
1. Pull the requested image (if needed)
2. Start a fresh container as the build agent
3. Run the steps inside that container
4. Destroy the container when the stage finishes

---

## Prerequisites

- Lab is up: `cd 99_misc/setup/docker && docker compose up -d --build`
- Jenkins is reachable at `http://localhost`
- You can log in to the Jenkins UI

Verify the controller can talk to the host Docker engine **before** touching Jenkins:

```sh
docker exec -it main docker info | head -20
docker exec -it main docker ps
```

You should see the host's running containers (including `main` itself). If you get `permission denied`, the socket mount or group permission is wrong — fix it before continuing.

---

## Setup Steps

### 1. Install the required plugins

`Manage Jenkins → Plugins → Available`

Install both:
- **Docker** (provides the cloud + agent template)
- **Docker Pipeline** (provides the `docker {}` agent block in Declarative Pipelines)

Restart Jenkins when prompted.

### 2. Add Docker as a Cloud

`Manage Jenkins → Clouds → New Cloud`

- Name: `docker-host`
- Type: `Docker`
- Docker Host URI: `unix:///var/run/docker.sock`
- Click **Test Connection** — it must return a Docker version string. If it errors, recheck step 1's verification commands.
- Enabled: ✅

### 3. Define a Docker Agent Template

Inside the cloud you just created, **Docker Agent Templates → Add**:

- Labels: `docker-agent`
- Enabled: 
- Name: `inbound-agent`
- Docker Image: `jenkins/inbound-agent:latest`
- Remote File System Root: `/home/jenkins/agent`
- Connect method: `Attach Docker container`
- Pull strategy: `Pull once and update latest`
- Usage: `Use this node as much as possible`

Save.

### 4. Free the controller from running builds (good hygiene)

`Manage Jenkins → Nodes → Built-In Node → Configure` → set **Number of executors** to `0`. This forces every build to land on a dynamic Docker agent.

---

## Practice

Create a new **Pipeline** job named `dynamic-agent-demo` and use the pipeline below. It must:

- Run with **no static agent** (`agent none` at the top)
- Execute the `Build` stage inside a `python:3.11-slim` container
- Execute the `Tools` stage inside a `node:20-slim` container
- Run both stages **in parallel** so two containers spin up at the same time
- Print the container hostname inside each stage to prove it is *not* the controller

### Acceptance criteria

- [ ] `docker ps` on the host shows two short-lived containers while the pipeline is running
- [ ] Both containers are gone after the build finishes (`docker ps -a` may still list them briefly until reaped)
- [ ] Console output shows two **different** hostnames, neither of which is `jenkins-main`
- [ ] Build result is `SUCCESS`

### Solution

```groovy
pipeline {
    agent none

    stages {
        stage('Parallel dynamic agents') {
            parallel {
                stage('Build (Python)') {
                    agent {
                        docker {
                            image 'python:3.11-slim'
                            label 'docker-agent'
                            args  '-u root'
                        }
                    }
                    steps {
                        sh 'hostname'
                        sh 'python3 --version'
                        sh 'pip install --quiet pytest && pytest --version'
                    }
                }
                stage('Tools (Node)') {
                    agent {
                        docker {
                            image 'node:20-slim'
                            label 'docker-agent'
                            args  '-u root'
                        }
                    }
                    steps {
                        sh 'hostname'
                        sh 'node --version'
                        sh 'npm --version'
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Build #${env.BUILD_NUMBER} finished — agents have been destroyed."
        }
    }
}
```

---

## Stretch Goals

1. **Pin the image** — replace `latest` with a specific digest (`python:3.11-slim@sha256:...`) and confirm Jenkins still launches the agent.
2. **Custom image** — write a tiny `Dockerfile` that bakes `pytest` into the image, push it locally (`docker build -t lab/python-tester .`), and use `image 'lab/python-tester'` so the install step disappears from the pipeline.
3. **Resource limits** — pass `args '--memory=512m --cpus=1'` and verify with `docker stats` while the build runs.

---

## Stretch Goal Solutions

### 1. Pin the image

First, look up the digest of the tag you want to pin:

```sh
docker pull python:3.11-slim
docker inspect --format='{{index .RepoDigests 0}}' python:3.11-slim
# python@sha256:<long-hex>
```

Then use the full digest reference in the pipeline:

```groovy
agent {
    docker {
        image 'python@sha256:<paste-digest-here>'
        label 'docker-agent'
        args  '-u root'
    }
}
```

After triggering the build, confirm Jenkins resolved the exact image:

```sh
docker exec main docker image inspect python@sha256:<digest> --format '{{.Id}}'
```

### 2. Custom image

Create `99_misc/setup/docker/Dockerfile.python-tester`:

```dockerfile
FROM python:3.11-slim
RUN pip install --no-cache-dir pytest pylint
WORKDIR /workspace
```

Build it on the **host** (the Jenkins controller shares the host's daemon, so the controller will see it immediately):

```sh
cd 99_misc/setup/docker
docker build -t lab/python-tester:1.0 -f Dockerfile.python-tester .
docker images | grep lab/python-tester
```

Use it in the pipeline — note `pytest` is now baked in, so no install step:

```groovy
pipeline {
    agent {
        docker {
            image 'lab/python-tester:1.0'
            label 'docker-agent'
            args  '-u root'
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'pytest --version'
                sh 'pylint --version'
            }
        }
    }
}
```

> [!] Tip: also set **Pull strategy → Never pull** on the agent template (or per-template) so Jenkins doesn't try to pull `lab/python-tester` from Docker Hub.

### 3. Resource limits

Add the limits via `args` and verify them while the build runs.

```groovy
pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
            label 'docker-agent'
            args  '-u root --memory=512m --cpus=1'
        }
    }
    stages {
        stage('Burn') {
            steps {
                sh 'cat /sys/fs/cgroup/memory.max || cat /sys/fs/cgroup/memory/memory.limit_in_bytes'
                sh 'nproc'
                sh 'python3 -c "import time; [time.sleep(0.01) for _ in range(500)]"'
            }
        }
    }
}
```

In a second terminal on the host **while the build is running**:

```sh
docker stats --no-stream
# look for the short-lived container — MEM LIMIT should show 512MiB and CPUs should be capped at 1
```

`nproc` inside the container will report `1`, and the cgroup file will read `536870912` (512 MiB).

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock` in the cloud test | Socket not mounted, or `main` container not running as a user that can read the socket. Check the `volumes:` block in `docker-compose.yml`. |
| `docker: command not found` inside a `sh` step on the controller | Only matters if you invoke `docker` directly. The dynamic-agent flow does not need it — the Docker plugin talks to the daemon over the socket. |
| Pipeline hangs on "Waiting for next available executor" | The controller has zero executors and no Docker agents are coming up. Recheck the cloud's **Test Connection** and the agent template label matches the pipeline's `label`. |
| Build fails with `permission denied` writing to the workspace | Add `args '-u root'` to the `docker {}` block (lab only) or set the agent template's user explicitly. |




