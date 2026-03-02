# Troubleshooting Guide

This document captures common issues encountered while building this CI pipeline and the exact checks/fixes used.

---

## 1) Jenkinsfile test fails: `checkout scm is only available...`

**Symptom**
- `ERROR: ‘checkout scm’ is only available when using “Multibranch Pipeline” or “Pipeline script from SCM”`

**Cause**
- The job was created as **Pipeline script** (pasted Jenkinsfile), so Jenkins had no SCM context and `scm` was undefined.

**Fix**
- Configure the job as **Pipeline script from SCM** and set:
  - Repository URL
  - Branch specifier (e.g., `*/main`)
  - Script Path = `Jenkinsfile`

---

## 2) Git checkout fails when using SSH: `Host key verification failed`

**Symptom**
- Checkout fails when repo URL looks like: `git@github.com:<user>/<repo>.git`

**Cause**
- SSH cloning requires the Jenkins runtime user to trust GitHub’s host key and have SSH key auth configured.

**Fix (simplest for public repos)**
- Switch Jenkins repository URL to HTTPS:
  - `https://github.com/<user>/<repo>.git`

---

## 3) Git fetch fails: `couldn't find remote ref refs/heads/main`

**Symptom**
- `fatal: couldn't find remote ref refs/heads/main`

**Cause**
- The branch ref did not exist on GitHub yet (commonly happens with an empty repo / no commits).

**Fix**
- Create at least one commit on GitHub (e.g., add `README.md`), then build `*/main`.

---

## 4) Jenkins checkout fails: `Could not resolve host: github.com`

**Symptom**
- `fatal: unable to access 'https://github.com/...': Could not resolve host: github.com`

**What this indicates**
- DNS resolution failed from the environment running the git command.

**Checks**
Run in WSL:
```bash
getent hosts github.com
ping -c 1 github.com
sudo -u jenkins getent hosts github.com

If you want a “service-like” check for the Jenkins user:

sudo systemd-run --uid=jenkins --pty /bin/bash -lc 'getent hosts github.com && curl -I https://github.com | head'

Fixes

Restart Jenkins to clear stale runtime state:

sudo systemctl restart jenkins

If DNS is unstable after reboot/VPN changes, consider pinning resolvers (advanced):

Disable WSL auto resolv.conf generation and set known DNS servers.

5) sudo -u jenkins ... fails with: failed to stat ... Permission denied

Symptom

fatal: failed to stat '/home/<user>/<project>': Permission denied

Cause

You ran a command as the jenkins user while your current directory was inside a path the jenkins user cannot access.

Fix
Run the command from a neutral directory like /tmp:

cd /tmp
sudo -u jenkins git ls-remote https://github.com/<user>/<repo>.git
6) Jenkins workspace directory missing or permission issues

Symptoms

/var/lib/jenkins/workspace missing

permission denied during checkout/build

Checks
Confirm Jenkins OS user exists:

id jenkins
getent passwd jenkins

Fix
Create and grant ownership/permissions:

sudo mkdir -p /var/lib/jenkins/workspace
sudo chown -R jenkins:jenkins /var/lib/jenkins/workspace
sudo chmod -R u+rwX /var/lib/jenkins/workspace
sudo -u jenkins bash -lc 'touch /var/lib/jenkins/workspace/_perm_test && rm /var/lib/jenkins/workspace/_perm_test'
7) Jenkins error: Unable to find Jenkinsfile

Cause

Jenkinsfile missing from the repo, or Script Path is wrong/case-mismatched.

Fix

Ensure the file is named exactly Jenkinsfile in the repo root.

In Jenkins job config: Script Path = Jenkinsfile.

8) Maven error in Jenkins: There is no POM in this directory

Symptom

Please verify you invoked Maven from the correct directory. -> ... no POM in this directory

Cause

Repo did not contain a pom.xml at the workspace root being built.

Fix

Commit a valid Maven project to the repo root:

pom.xml

src/main/...

src/test/...

9) Git push fails: GitHub password authentication not supported

Symptom

GitHub rejects HTTPS password auth for git operations.

Fix

Use a GitHub Personal Access Token (PAT) as the “password” for HTTPS pushes.

Optional: store credentials in WSL to avoid retyping:

git config --global credential.helper store

Verify an entry exists (prints line number, not the secret):

grep -n "github.com" ~/.git-credentials
10) Git push rejected: non-fast-forward / divergent histories

Symptom

rejected (fetch first) or warnings about divergent branches.

Cause

Remote repo had commits not present locally (e.g., files created in GitHub UI).

Fix (merge approach)

git pull origin main --allow-unrelated-histories --no-rebase
# resolve conflicts if prompted
git push origin main

If a conflict occurs for a file like Jenkinsfile:

Choose one side, git add <file>, then git commit to conclude the merge.

11) Quality Gate times out (waitForQualityGate)

Symptom

Quality Gate stage times out while Sonar task is still IN_PROGRESS.

Cause

SonarQube → Jenkins webhook not configured or not reachable from the SonarQube container.

Fix

Find docker bridge IP:

ip -4 addr show docker0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}'

Test container → Jenkins connectivity:

docker exec -it sonarqube bash -lc "timeout 3 bash -c '</dev/tcp/172.17.0.1/8080' && echo CONNECT_OK || echo CONNECT_FAIL"

Configure SonarQube webhook:

URL: http://172.17.0.1:8080/sonarqube-webhook/

12) Smoke test fails: no main manifest attribute

Symptom

no main manifest attribute, in target/<app>.jar

Cause

The jar produced is not a Spring Boot executable jar.

Fix

Ensure Spring Boot packaging is configured correctly in pom.xml:

Use spring-boot-starter-parent

Configure spring-boot-maven-plugin with repackage

Explicitly set the application main class if needed, e.g.:

com.example.demo.DemoApplication

13) Declarative pipeline syntax errors (e.g., Unknown stage section "always")

Cause

Incorrect Declarative Pipeline structure (e.g., always {} placed outside post {}) or missing braces.

Fix

In Declarative Pipeline, always must be under post:

post {
  always {
    // steps
  }
}

Also ensure every stage { ... } block closes before the next stage begins.