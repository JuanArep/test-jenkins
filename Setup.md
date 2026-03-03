# Environment Setup (Windows + WSL): Jenkins + Docker + SonarQube

This guide reproduces the environment used by the CI pipeline in this repo:
- Jenkins (CI server) on WSL Ubuntu
- Docker Engine on WSL Ubuntu
- SonarQube running as a Docker container

> Assumptions:
> - You are using WSL2 on Windows.
> - Commands are for Ubuntu on WSL.
> - Jenkins UI will be reachable at http://localhost:8080
> - SonarQube UI will be reachable at http://localhost:9000

---

## 1) Start Ubuntu on WSL (Windows)

List installed WSL distributions and versions:
```powershell
wsl --list --verbose
```
Why: Confirms which distros exist and whether they are running on WSL2.

Start your CI distro:
```powershell
wsl -d <YOUR_DISTRO_NAME>
```
Why: Opens the Linux environment where Jenkins/Docker/SonarQube will run.

## 2) Update Ubuntu packages (inside WSL)
```bash
sudo apt update && sudo apt -y upgrade
```
Why:

apt update refreshes the package index (available versions).

apt upgrade installs security/bug fixes and reduces install issues later.

## 3) Install base utilities
```bash
sudo apt -y install ca-certificates curl gnupg lsb-release unzip git
```
Why:

ca-certificates: required for HTTPS/TLS certificate validation.

curl: download files and test HTTP endpoints.

gnupg: verify signatures for repositories/packages.

lsb-release: identifies Ubuntu version (useful for debugging and installers).

unzip: extract downloaded archives.

git: required to clone/push code.

## 4) Install Java + Maven (build toolchain)
```bash 
sudo apt -y install openjdk-17-jdk maven
java -version
mvn -version
```
Why:

openjdk-17-jdk: builds and runs the Spring Boot app (Java 17).

maven: reads pom.xml, downloads dependencies, runs tests, packages the jar.

version checks confirm the toolchain matches what the pipeline expects.

## 5) Enable systemd in WSL (required for services)

Check what PID 1 is:
```bash
ps -p 1 -o comm=
```
Why: Jenkins and Docker are typically managed by systemd. If PID 1 isn’t systemd, service control can fail.

Enable systemd for WSL:
```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true
EOF
```
Restart WSL (Windows PowerShell):
```powershell
wsl --shutdown
```
Why: Applies the WSL systemd setting by restarting the VM.

After restart, confirm systemd is active:
```bash
ps -p 1 -o comm=
```
Expected output: systemd

6) Install Docker Engine (WSL)

Install Docker:
```bash
sudo apt -y install docker.io
docker --version
```
Why: Docker runs SonarQube reliably without manual native installation.

Enable/start Docker service:
```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker --no-pager
```
Why: Docker requires the background daemon (dockerd) running.

Allow your user to run Docker without sudo:
```bash
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
Why:

Adds your user to the docker group so docker commands don’t require root.

newgrp applies the group membership immediately.

docker ps validates permissions (no “permission denied”).

7) Run SonarQube using Docker

Create persistent volumes:
```bash
docker volume create sonarqube_data
docker volume create sonarqube_extensions
docker volume create sonarqube_logs
```
Why: SonarQube stores data/plugins/logs; volumes persist them across container restarts.

Run SonarQube:
```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  -v sonarqube_logs:/opt/sonarqube/logs \
  sonarqube:lts
```
Why:

-d: run in background

--name sonarqube: stable container name

-p 9000:9000: exposes the SonarQube UI

-v ...: persists SonarQube data/extensions/logs

sonarqube:lts: stable long-term support image

Verify it’s running:

docker ps --filter "name=sonarqube"

Access UI:

http://localhost:9000

## 8) Install Jenkins (CI server)

Install dependency:
```bash
sudo apt -y install fontconfig
```
Why: Some Jenkins/Java setups require font libraries even in headless environments.

Install Jenkins and start it:
```bash
sudo apt -y install jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins --no-pager
```
Why:

Jenkins runs as a background service.

enable starts it after reboot.

status confirms it is running.

Unlock Jenkins (first-time setup):
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Why: Required for first-time admin setup in the Jenkins UI.

Access UI:

http://localhost:8080

## 9) Configure SonarQube → Jenkins webhook (Quality Gate reliability)

Quality Gate (waitForQualityGate) is most reliable when SonarQube can notify Jenkins that analysis is complete.

Find Docker bridge IP (host address reachable from containers):
```bash
ip -4 addr show docker0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'
```
Why: SonarQube runs inside a container; localhost in the container is not the host. The docker0 bridge IP (often 172.17.0.1) is reachable from containers.

Connectivity check from the SonarQube container to Jenkins (without curl):
```bash
docker exec -it sonarqube bash -lc "timeout 3 bash -c '</dev/tcp/172.17.0.1/8080' && echo CONNECT_OK || echo CONNECT_FAIL"
```
Why: Confirms the container can reach Jenkins on port 8080.

Create webhook in SonarQube UI:

SonarQube → Administration → Configuration → Webhooks → Create

URL:

http://172.17.0.1:8080/sonarqube-webhook/

Why: Jenkins receives completion notifications so the Quality Gate step doesn’t time out.

## 10) After reboot: quick health check
```bash
sudo systemctl status jenkins --no-pager
sudo systemctl status docker --no-pager
docker ps --filter "name=sonarqube"
java -version
mvn -version
```
Why: Confirms services and toolchain are running before starting builds.

```
```

---