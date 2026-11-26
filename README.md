# Jenkins-lab
# Jenkins-lab: CI/CD Pipeline with Jenkins, Docker and Kubernetes

A hands-on DevOps project built entirely on a local machine (Windows, WSL Ubuntu, Docker Desktop). A small Python app goes from a GitHub commit to a tested, versioned container image on Docker Hub, and runs on Kubernetes behind a Service.

## How it works

```
Developer -> GitHub -> Jenkins (in Docker) -> Docker Hub -> Kubernetes (Docker Desktop)
                          |                                       |
                  test, build, push                  Deployment + Service
```

Every Jenkins build runs this pipeline, defined as code in the `Jenkinsfile`:

| Stage | What happens |
|---|---|
| Checkout | Jenkins pulls the latest code from GitHub |
| Test | `pytest` runs the unit tests. A failure stops the pipeline |
| Build Image | Docker builds the image, tagged with the build number and `latest` |
| Run Image | The app is started once as a smoke test |
| Push Image | The image is pushed to Docker Hub using a stored Jenkins credential |

Kubernetes then runs the image:

- **Deployment** (`k8s-deployment.yaml`): keeps the desired number of pods running and replaces any that die.
- **Service** (`k8s-service.yaml`): gives the pods one stable address and spreads traffic across them. It finds pods by the label `app: jenkins-lab`.

## Tech stack

Git and GitHub, Jenkins (declarative pipeline, running in a custom Docker image), Docker and Docker Hub, Kubernetes (Docker Desktop cluster), WSL2 Ubuntu, VS Code.

## Repository contents

| File | Purpose |
|---|---|
| `app.py` | Small Python app |
| `test_app.py` | Unit test run by the pipeline |
| `Dockerfile` | Builds the application image (runs a web server on port 8000) |
| `Jenkinsfile` | The pipeline definition (pipeline as code) |
| `k8s-deployment.yaml` | Kubernetes Deployment |
| `k8s-service.yaml` | Kubernetes Service (ClusterIP) |

## Run it yourself

1. Start Jenkins with Docker access:
   ```
   docker run -d --name jenkins -u root -p 8080:8080 -p 50000:50000 \
     -v jenkins_home:/var/jenkins_home \
     -v /var/run/docker.sock:/var/run/docker.sock my-jenkins
   ```
   (`my-jenkins` is `jenkins/jenkins:lts-jdk17` plus `docker.io`, `python3` and `python3-pytest`.)
2. Add a Jenkins credential with the ID `dockerhub` (Docker Hub username and an access token).
3. Create a Pipeline job ("Pipeline script from SCM") pointing at this repo, then click **Build Now**.
4. Deploy to the Kubernetes cluster built into Docker Desktop:
   ```
   kubectl apply -f k8s-deployment.yaml
   kubectl apply -f k8s-service.yaml
   kubectl rollout restart deployment jenkins-lab
   kubectl port-forward service/jenkins-lab 8000:8000
   ```
   Then `curl -I http://localhost:8000` returns `HTTP/1.0 200 OK`.

## What I learned

Kubernetes ideas I tested by hand:
- **Self-healing:** deleting a pod makes the Deployment start a replacement, because it compares the desired state (`replicas: 1`) with what is running.
- **Scaling:** `kubectl scale --replicas=3` runs three copies of the same image.
- **Services and labels:** a Service reaches pods through a label selector, not through ports or file names.
- **ClusterIP** addresses are internal to the cluster, so `kubectl port-forward` is needed to test from outside.

Problems I hit and fixed:
- **WSL DNS failure** (`Could not resolve host`): diagnosed in layers with `ping 8.8.8.8` vs `ping google.com`.
- **Git "Author identity unknown"**: set `user.name` and `user.email`, then used a personal access token.
- **Jenkinsfile syntax error** (`expecting '}'`): an unbalanced brace.
- **Pytest `NameError`**: a stray line in `app.py`, caught by the pipeline before anything was built.
- **"Could not find credentials entry"**: the credential ID must match the Jenkinsfile exactly.
- **Git "not in a git directory"**: a file-ownership problem; a setting made inside a container was lost when the container was recreated, so the fix belongs in the image.
- **`kubectl` talking to Jenkins**: with no cluster configured it defaulted to `localhost:8080`.
- **`CrashLoopBackOff`**: first the container exited after printing, then the Dockerfile `CMD` used single quotes, which is invalid JSON syntax.
- **"Cannot connect to the Docker daemon"**: the Jenkins container lost its Docker socket mount.

## Known limitations and next steps

- The pipeline builds and pushes the image, but the Kubernetes redeploy is still a manual `kubectl` step. Adding a Deploy stage to the Jenkinsfile is next.
- The Deployment uses the `latest` tag. Deploying a specific build number would make each release traceable.
- Planned: a multi-service app with a database, Helm, Terraform, GitHub Actions, GitOps with ArgoCD, and monitoring with Prometheus and Grafana.

## Mapping to the cloud

Built locally, but the concepts translate directly. Docker Hub maps to ECR or ACR, the local cluster to EKS or AKS, a ClusterIP or `LoadBalancer` Service to a cloud load balancer, and Jenkins credentials to Secrets Manager or Key Vault.
