# GitOps Spring Boot Example

A minimal Spring Boot app with a GitOps-friendly workflow and step-by-step VM instructions.

## What is included

- `pom.xml` and Spring Boot source code
- `Dockerfile` for container image build
- Kubernetes manifests in `manifests/k8s`
- `Jenkinsfile` for CI pipeline
- `manifests/jenkins/jenkins-deployment.yaml` (optional in-cluster Jenkins)
- `manifests/argocd/application.yaml` (Argo CD application manifest)
- no bootstrap scripts are included here; you will write those inside the Ubuntu VM as a learning exercise

## GitOps pattern (high level)

1. App source, Dockerfile and Kubernetes manifests are stored in Git.
2. Jenkins (CI) builds the app and produces a Docker image.
3. Jenkins pushes the image to a registry accessible by the cluster (or loads it into Minikube).
4. Argo CD (CD) watches the manifests in Git and synchronizes the desired state into Kubernetes.

## VM (Ubuntu) method using Minikube — step-by-step

This repo includes a complete workflow to run everything inside a single Ubuntu VM (good for VMware Workstation). Follow these steps inside the VM.

1. Prepare VM resources: 4 CPU, 8 GB RAM, 40+ GB disk recommended.
2. Copy or clone this repository into the VM.
3. Manual VM workflow — run commands directly inside the Ubuntu VM

This repository does not include any bootstrap scripts. You will run the setup commands one by one inside the VM so you can see exactly what happens at each step.

Recommended manual steps in the VM:

1. Install Docker and required system tools:
   ```bash
   sudo apt update
   sudo apt install -y docker.io conntrack socat curl
   sudo usermod -aG docker $USER
   newgrp docker
   ```

2. Install `kubectl`:
   ```bash
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   chmod +x kubectl
   sudo mv kubectl /usr/local/bin/
   ```

3. Install `minikube`:
   ```bash
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube
   ```

4. Install Java 17 and Maven:
   ```bash
   sudo apt install -y openjdk-17-jdk maven git
   ```

5. Start Minikube with Docker driver:
   ```bash
   minikube start --driver=docker --cpus=4 --memory=8192
   ```

6. Enable Minikube registry and Argo CD:
   ```bash
   minikube addons enable registry
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

7. Find the local registry URL for Jenkins to push images:
   ```bash
   minikube service registry -n kube-system --url
   ```

8. Use that `REGISTRY` value in your Jenkins pipeline or job configuration.

### Quick deployment flow

- Jenkins builds the Docker image and pushes it to the Minikube registry using the configured `REGISTRY`.
- Update `manifests/k8s/deployment.yaml` to point to the image built by Jenkins.
- Argo CD observes the manifests in the Git repo and syncs them into Minikube.

## Running Jenkins in the VM

- If you deploy Jenkins inside the VM, you can run it as a Docker container and mount `/var/run/docker.sock:/var/run/docker.sock` so Jenkins can build and push images.
- You can also build a custom Jenkins image inside the VM that includes Docker CLI, Maven and Git.

For persistent storage in production, replace the `emptyDir` volumes in `manifests/jenkins/jenkins-deployment.yaml` with a PVC or an external volume.

## Using Argo CD

- Install Argo CD manually into Minikube using the commands above.
- You can create an Argo CD `Application` (we provide `manifests/argocd/application.yaml`) that points to this repo and the `manifests/k8s` path.

## Example Jenkinsfile notes

- `Jenkinsfile` in this repo contains stages:
  - Checkout
  - Set image name using the build number
  - Build (`mvn package`)
  - Unit tests
  - Docker image build
  - Push image to `REGISTRY`
  - Update `manifests/k8s/deployment.yaml` to reference the new image tag
  - Commit and push the manifest change back to Git

- This is the GitOps signal: Jenkins updates the manifest in Git so Argo CD can detect the new image and apply it.
- You must configure a Jenkins credential named `github-creds` (username/password) or adapt `GIT_CREDENTIALS_ID` to your credential ID.

Set the pipeline environment variable `REGISTRY` to the value returned by `minikube service registry -n kube-system --url`.
