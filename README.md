# GitOps Spring Boot Example

A minimal Spring Boot app with a GitOps-friendly workflow and step-by-step VM instructions.

## What is included

- `pom.xml` and Spring Boot source code
- `Dockerfile` for container image build
- Kubernetes manifests in `manifests/k8s`
- `Jenkinsfile` for CI pipeline
- `manifests/jenkins/jenkins-deployment.yaml` (optional in-cluster Jenkins)
- `manifests/argocd/application.yaml` (Argo CD application manifest)
- bootstrap helpers in `scripts/` (`setup-ubuntu-vm.sh` for a VM-based flow)

## GitOps pattern (high level)

1. App source, Dockerfile and Kubernetes manifests are stored in Git.
2. Jenkins (CI) builds the app and produces a Docker image.
3. Jenkins pushes the image to a registry accessible by the cluster (or loads it into Minikube).
4. Argo CD (CD) watches the manifests in Git and synchronizes the desired state into Kubernetes.

## VM (Ubuntu) method using Minikube — step-by-step

This repo includes a complete workflow to run everything inside a single Ubuntu VM (good for VMware Workstation). Follow these steps inside the VM.

1. Prepare VM resources: 4 CPU, 8 GB RAM, 40+ GB disk recommended.
2. Copy or clone this repository into the VM.
3. Learning exercises — scripts intentionally omitted

This repository intentionally excludes bootstrap scripts. The goal is for you to clone the repo inside your Ubuntu VM and implement the bootstrap steps yourself so you learn each command and why it is used.

Suggested exercises to implement inside the VM:

- `setup-ubuntu-vm.sh` — install Docker, `kubectl`, `minikube`, Helm, Java 17 and Maven. Tasks:
  - install prerequisites (`docker.io`, `conntrack`, `socat`),
  - install `kubectl` and `minikube`,
  - install Java 17 and Maven,
  - print instructions for starting Minikube.

- `bootstrap-minikube.sh` — start Minikube (driver=docker), enable a local registry addon, and install Argo CD into the cluster.

- `jenkins/Dockerfile` — (optional) build a Jenkins image that includes Docker CLI, Maven and Git so Jenkins can build and push images.

Example commands to run in your VM after you create the scripts:

```bash
chmod +x setup-ubuntu-vm.sh
./setup-ubuntu-vm.sh

# start minikube
minikube start --driver=docker --cpus=4 --memory=8192

# enable registry and install Argo CD
minikube addons enable registry
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

After you complete the exercises inside the VM:

- Jenkins UI will be reachable at the port you expose on the VM (for example `http://localhost:8081`).
- To find the Minikube registry URL to use in your Jenkins pipeline, run:

```bash
minikube service registry -n kube-system --url
```

- Use the returned URL as the `REGISTRY` in your `Jenkinsfile` or Jenkins job.

### Quick deployment flow

- Jenkins builds the Docker image and pushes it to the Minikube registry (configured by `REGISTRY`).
- Update `manifests/k8s/deployment.yaml` to point to the image built by Jenkins (or let Jenkins update the manifest).
- Argo CD observes the manifests in the Git repo and syncs them into Minikube.

## Running Jenkins in the VM

- The bootstrap builds and runs a `jenkins-docker` image which includes `docker` and `maven` so your pipeline can build and push images.
- The Jenkins container is run with `-v /var/run/docker.sock:/var/run/docker.sock` so it can use the VM Docker daemon.

For persistent storage in production, replace the `emptyDir` volumes in `manifests/jenkins/jenkins-deployment.yaml` with a PVC or an external volume.

## Using Argo CD

- Argo CD is installed into Minikube by the bootstrap script.
- You can create an Argo CD `Application` (we provide `manifests/argocd/application.yaml`) that points to this repo and the `manifests/k8s` path.

## Example Jenkinsfile notes

- `Jenkinsfile` in this repo contains stages:
  - Checkout
  - Build (`mvn package`)
  - Unit tests
  - Docker image build
  - Push image to `REGISTRY`
  - Update `manifests/k8s/deployment.yaml` to reference the new image tag

Set the pipeline environment variable `REGISTRY` to the value returned by `minikube service registry -n kube-system --url`.

## Next steps I can help with

- Walk you step-by-step through running the bootstrap inside your VM and validating each step.
- Provide a PowerShell helper if you want to automate VM file transfer/startup from Windows.
- Convert the Jenkins pipeline to use `minikube image load` instead of a registry (if you prefer that approach).

---

If you want to continue now, tell me if I should:

1) Walk you interactively through running `./scripts/setup-ubuntu-vm.sh` inside your Ubuntu VM and checking outputs step-by-step, or
2) Produce a PowerShell helper script to set up the VM and copy files from Windows into it automatically.

