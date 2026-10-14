# Runbook: Adding a Jenkins SSH Agent Node

Scope: one permanent Linux VM (Ubuntu, Azure) added as a Jenkins agent via **SSH**, then used by pipelines.
Replace every `<placeholder>` before running anything.

---

## 0. How it works

```
 Jenkins controller  ──── SSH (TCP 22) ────►  Agent VM
 (opens connection,                           (jenkins user, Java,
  copies remoting.jar,                         workdir, Docker/tools)
  runs builds remotely)
```

- The **controller** initiates everything. The agent never connects back.
- The controller logs in as `jenkins@<agent>` with a private key stored in Jenkins Credentials.
- Jenkins copies `remoting.jar` to the agent on every launch, so there is no manual agent.jar to maintain.

| Side | What you do there |
|---|---|
| Azure network | Open 22 controller → agent only |
| Agent VM | Java, `jenkins` user, workdir, SSH access, build tools |
| Controller VM | Plugin, key pair, `known_hosts` (as jenkins user), manual SSH test |
| Jenkins UI | Credential, node, labels, executors |
| Pipeline | Target the node by label |

---

## 1. Controller prerequisites (once per controller)

**1.1 Jenkins URL.** Manage Jenkins → System → **Jenkins URL** = `https://jenkins.<domain>/`

**1.2 Plugins.** Manage Jenkins → Plugins → install if missing:

| Plugin | Why |
|---|---|
| SSH Build Agents | Provides "Launch agents via SSH" |
| Credentials, SSH Credentials | Store the private key |
| Pipeline, Pipeline: Declarative | Jenkinsfiles |
| Git, GitHub Branch Source | Checkout / multibranch |
| Credentials Binding | `withCredentials` |
| Docker Pipeline | `agent { docker { ... } }` (optional) |
| Workspace Cleanup | `cleanWs()` |
| Timestamper | `timestamps()` |

**1.3 Stop builds on the controller.** Manage Jenkins → Nodes → Built-In Node → Configure → **Number of executors = 0**

**1.4 Find which user and home Jenkins runs as.** Needed for steps 4–5.

| Install type | User | JENKINS_HOME |
|---|---|---|
| apt package on VM | `jenkins` | `/var/lib/jenkins` |
| Docker `jenkins/jenkins` | `jenkins` (uid 1000) | `/var/jenkins_home` |
| Kubernetes (Helm) | `jenkins` (uid 1000) | `/var/jenkins_home` |

```bash
# VM install: confirm
ps -o user= -p $(pgrep -f jenkins.war) ; getent passwd jenkins | cut -d: -f6
```

**1.5 Backup** `JENKINS_HOME` before changing nodes.

---

## 2. Azure network

**Agent VM NSG (inbound)**
| Port | Source | Purpose |
|---|---|---|
| 22/TCP | Controller private IP (`<controller-ip>/32`) **only** | Jenkins SSH launch |

**Agent VM NSG (outbound)**
| Port | Destination | Purpose |
|---|---|---|
| 443/TCP | Internet / service tags | GitHub, Docker Hub, apt mirrors, package registries |

**Controller NSG (outbound)**
| Port | Destination | Purpose |
|---|---|---|
| 22/TCP | Agent private IP | SSH to agent |

Notes:
- Keep controller ↔ agent traffic on **private IPs** (same VNet or peered VNets).
- Port 50000 is **not** needed for SSH agents.
- Do not open 22 to the internet. Admin access goes through Bastion or a VPN.

---

## 3. Agent VM preparation (run on the agent)

**3.1 Packages**
```bash
sudo apt update
# Java version must be one the controller supports (Manage Jenkins → System Information → java.version)
sudo apt install -y openjdk-17-jre-headless git openssh-server chrony curl unzip
sudo systemctl enable --now ssh chrony
java -version
```

**3.2 jenkins user and workdir**
```bash
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /home/jenkins/agent
sudo chown -R jenkins:jenkins /home/jenkins
sudo passwd -l jenkins          # no password login; key only
```

**3.3 Build tools (only what your pipelines need)**
```bash
# Docker
sudo apt install -y docker.io
sudo usermod -aG docker jenkins
sudo systemctl enable --now docker

# Example: Node.js / other toolchains as required
# sudo apt install -y nodejs npm
```

**3.4 sshd must allow the jenkins user with key auth** (hardened VMs often block it)
```bash
sudo sshd -T | grep -Ei 'allowusers|allowgroups|denyusers|pubkeyauthentication|passwordauthentication'
```
- `pubkeyauthentication yes` must be present.
- If `AllowUsers` or `AllowGroups` is set, add `jenkins`. Edit `/etc/ssh/sshd_config` or the drop-in under `/etc/ssh/sshd_config.d/`:
  ```
  AllowUsers <existing-users> jenkins
  ```
- Then validate and reload:
  ```bash
  sudo sshd -t && sudo systemctl reload ssh
  ```

**3.5 Disk and housekeeping**
```bash
df -h /home /var/lib/docker      # keep >= 20 GB free
# weekly Docker cleanup
echo '0 3 * * 0 root docker system prune -af --filter "until=168h"' | sudo tee /etc/cron.d/docker-prune
```

**3.6 Note the agent's private IP**
```bash
hostname -I
```

---

## 4. SSH key pair (controller side → agent side)

Generate the key **on the controller as the Jenkins user**, so ownership and paths are correct.

**4.1 On the controller**
```bash
# VM install
sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh
sudo -u jenkins ssh-keygen -t ed25519 -N "" -C "jenkins-agent-<agent-name>" \
  -f /var/lib/jenkins/.ssh/agent_<agent-name>
sudo cat /var/lib/jenkins/.ssh/agent_<agent-name>.pub      # copy this

# Docker controller
docker exec -u jenkins <jenkins-container> sh -c \
  'mkdir -p ~/.ssh && ssh-keygen -t ed25519 -N "" -C jenkins-agent-<agent-name> -f ~/.ssh/agent_<agent-name> && cat ~/.ssh/agent_<agent-name>.pub'

# Kubernetes controller
kubectl -n <jenkins-ns> exec -it <jenkins-pod> -- sh -c \
  'mkdir -p ~/.ssh && ssh-keygen -t ed25519 -N "" -C jenkins-agent-<agent-name> -f ~/.ssh/agent_<agent-name> && cat ~/.ssh/agent_<agent-name>.pub'
```

**4.2 On the agent: install the public key**
```bash
sudo -u jenkins mkdir -p /home/jenkins/.ssh
echo '<paste-public-key>' | sudo -u jenkins tee -a /home/jenkins/.ssh/authorized_keys
sudo chmod 700 /home/jenkins/.ssh
sudo chmod 600 /home/jenkins/.ssh/authorized_keys
sudo chown -R jenkins:jenkins /home/jenkins/.ssh
```

---

## 5. Trust the agent's host key (controller, as the Jenkins user)

```bash
# VM install
sudo -u jenkins sh -c 'ssh-keyscan -H <agent-ip> >> /var/lib/jenkins/.ssh/known_hosts'
sudo chmod 700 /var/lib/jenkins/.ssh
sudo chmod 644 /var/lib/jenkins/.ssh/known_hosts

# Docker controller (persists because /var/jenkins_home is the volume)
docker exec -u jenkins <jenkins-container> sh -c 'ssh-keyscan -H <agent-ip> >> ~/.ssh/known_hosts'

# Kubernetes controller (persists if JENKINS_HOME is on a PVC)
kubectl -n <jenkins-ns> exec -it <jenkins-pod> -- sh -c 'ssh-keyscan -H <agent-ip> >> ~/.ssh/known_hosts'
```
Running `ssh-keyscan` as root or your own user writes the wrong file, and the launch then fails with `Host key verification failed`.

Optional, stronger check: compare the fingerprint with the one the agent itself reports.
```bash
# on agent
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# on controller
sudo -u jenkins ssh-keygen -lF <agent-ip> -f /var/lib/jenkins/.ssh/known_hosts
```

---

## 6. Manual SSH test from the controller (must pass before step 7)

```bash
# VM install
sudo -u jenkins ssh -i /var/lib/jenkins/.ssh/agent_<agent-name> -o BatchMode=yes \
  jenkins@<agent-ip> 'whoami && java -version && docker ps >/dev/null && echo DOCKER_OK'

# Docker controller
docker exec -u jenkins <jenkins-container> ssh -i ~/.ssh/agent_<agent-name> -o BatchMode=yes \
  jenkins@<agent-ip> 'whoami && java -version'
```
Expected: no password prompt, prints `jenkins`, the Java version and `DOCKER_OK`.
If `docker ps` fails here, log out and back in. The group change applies to new sessions only.

---

## 7. Add the credential (Jenkins UI)

Manage Jenkins → Credentials → System → Global credentials → **Add Credentials**
- Kind: **SSH Username with private key**
- Scope: Global
- ID: `ssh-agent-<agent-name>`
- Description: `SSH key for <agent-name>`
- Username: `jenkins`
- Private Key: **Enter directly** → paste the full contents of `agent_<agent-name>`, including the BEGIN/END lines:
  ```bash
  sudo cat /var/lib/jenkins/.ssh/agent_<agent-name>
  ```
- Passphrase: empty
- Create.

Then remove the private key file from disk (Jenkins now holds it encrypted):
```bash
sudo shred -u /var/lib/jenkins/.ssh/agent_<agent-name>
```

---

## 8. Create the node (Jenkins UI)

Manage Jenkins → Nodes → **New Node** → Name `<agent-name>` → **Permanent Agent** → Create

| Field | Value |
|---|---|
| Description | `Ubuntu build VM, Docker` |
| Number of executors | `2` (start ≈ vCPUs / 2) |
| Remote root directory | `/home/jenkins/agent` |
| Labels | `linux docker` (space-separated) |
| Usage | **Only build jobs with label expressions matching this node** |
| Launch method | **Launch agents via SSH** |
| → Host | `<agent-ip>` (private IP) |
| → Credentials | `jenkins (SSH key for <agent-name>)` |
| → Host Key Verification Strategy | **Known hosts file Verification Strategy** |
| → Advanced → Port | `22` |
| → Advanced → JavaPath | blank (or `/usr/lib/jvm/java-17-openjdk-amd64/bin/java` if multiple JDKs) |
| → Advanced → Connection Timeout / Retries | `60` / `10` / wait `15` |
| Availability | **Keep this agent online as much as possible** |
| Node Properties (optional) | Environment variables, e.g. `DOCKER_BUILDKIT=1` |

Save.

---

## 9. Verify the node

1. Manage Jenkins → Nodes → `<agent-name>` should show online (no red X).
2. `<agent-name>` → **Log** should end with:
   `Agent successfully connected and online`
3. On the agent, confirm remoting was copied:
   ```bash
   ls -l /home/jenkins/agent/remoting.jar
   ```
4. Quick job: New Item → Pipeline → script:
   ```groovy
   pipeline {
     agent { label 'linux && docker' }
     stages {
       stage('Smoke') {
         steps { sh 'hostname && whoami && java -version && docker version' }
       }
     }
   }
   ```
   The console must show `Running on <agent-name> in /home/jenkins/agent/workspace/...`.

---

## 10. Build pipelines on this node

**10.1 Credentials used by the pipeline** (Manage Jenkins → Credentials):
- `dockerhub`: Username with password (Docker Hub user + **access token**, not your password)
- `github`: Username with password (GitHub user + PAT) for private repos

**10.2 Create the job**
1. New Item → `<app>` → **Multibranch Pipeline**
2. Branch Sources → GitHub → credentials `github` → repo URL
3. Build Configuration → by Jenkinsfile → `Jenkinsfile`
4. Scan triggers → Periodically, 15 min (fallback if no webhook)
5. Optional webhook: GitHub repo → Settings → Webhooks → `https://<jenkins-url>/github-webhook/`, JSON, push events. The controller must accept 443 from GitHub's hook IPs.

**10.3 Jenkinsfile (in the app repo)**
```groovy
pipeline {
  agent { label 'linux && docker' }        // runs on this SSH agent

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    disableConcurrentBuilds()
    timeout(time: 30, unit: 'MINUTES')
  }

  environment {
    IMAGE = "docker.io/<dockerhub-user>/<app>"
    TAG   = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Test') {
      agent {
        docker {
          image 'node:20-alpine'           // swap for your stack
          reuseNode true                   // same agent + workspace
        }
      }
      steps { sh 'npm ci && npm test' }
    }

    stage('Build image') {
      steps { sh 'docker build -t $IMAGE:$TAG -t $IMAGE:latest .' }
    }

    stage('Push image') {
      when { branch 'main' }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub',
                         usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
            docker push $IMAGE:latest
            docker logout
          '''
        }
      }
    }
  }

  post {
    always  { sh 'docker image rm $IMAGE:$TAG $IMAGE:latest || true' }
    cleanup { cleanWs() }
  }
}
```

**10.4 Pin to this exact node** (instead of any node with the label):
```groovy
agent { node { label '<agent-name>' } }
```
For freestyle jobs, use Configure → **Restrict where this project can be run** → `<agent-name>` or `linux && docker`.

---

## 11. Operations

| Task | How |
|---|---|
| Add another SSH agent | Repeat steps 2–9 with a new name/IP. Steps 4 and 7 can reuse the same key if policy allows. |
| Take offline for maintenance | Node → **Mark this node temporarily offline** (running builds finish) |
| Docker group changed on agent | Node → **Disconnect**, then **Launch agent** (new SSH session picks up the group) |
| Agent VM rebuilt (new host key) | Controller: `sudo -u jenkins ssh-keygen -R <agent-ip> -f /var/lib/jenkins/.ssh/known_hosts`, then redo step 5 |
| Rotate key | New key pair (step 4) → add new pub key → update credential (step 7) → relaunch → remove old line from `authorized_keys` |
| Controller upgraded | Nothing on agent; remoting.jar is re-copied on next launch. Check Java compatibility. |
| Remove agent | Mark offline → Delete Agent → on VM remove the controller key from `authorized_keys` → close NSG rule |

---

## 12. Troubleshooting

| Symptom (node log / console) | Cause → Fix |
|---|---|
| `Host key verification failed` | `known_hosts` missing or written as the wrong user → redo step 5 as the Jenkins user |
| `Authentication failed` / `Permission denied (publickey)` | Pub key not in `authorized_keys`, wrong perms (700/600), wrong username in credential, or sshd `AllowUsers` blocks `jenkins` |
| `Connection timed out` | NSG/firewall: 22 from controller IP not allowed, or using the public IP instead of the private one |
| `Connection refused` | sshd not running on agent: `sudo systemctl status ssh` |
| `java: command not found` / `Java version X not supported` | Install the correct JDK on the agent or set JavaPath (step 8) |
| Node online but builds stay queued: `There are no nodes with the label ‘x’` | Label mismatch between node and Jenkinsfile |
| `docker: permission denied while trying to connect to the Docker daemon socket` | `jenkins` not in `docker` group, or agent not relaunched after `usermod` |
| `No space left on device` | Clean workspaces/images; enlarge disk; check the cron in 3.5 |
| Agent drops repeatedly | Agent VM memory/OOM (`dmesg`), or network idle timeout. Raise retries in step 8 |

Useful commands:
```bash
# agent
sudo journalctl -u ssh -f                    # watch login attempts while launching
sudo tail -f /var/log/auth.log
# controller
sudo -u jenkins ssh -vvv -i <key> jenkins@<agent-ip>   # verbose SSH debug
```

---

## 13. Checklist

**Controller**
- [ ] Jenkins URL set; SSH Build Agents (+ pipeline plugins) installed
- [ ] Built-in node executors = 0
- [ ] Key pair generated as the Jenkins user
- [ ] `known_hosts` populated **as the Jenkins user**
- [ ] Manual `ssh -o BatchMode=yes` test passes
- [ ] Private key file shredded after adding the credential

**Network**
- [ ] Agent NSG: 22 inbound from controller IP only
- [ ] Controller NSG: 22 outbound to agent
- [ ] Agent outbound 443 (GitHub, Docker Hub, apt)

**Agent VM**
- [ ] Java (compatible), git, openssh-server, chrony installed
- [ ] `jenkins` user, `/home/jenkins/agent`, password locked
- [ ] Docker installed, `jenkins` in `docker` group
- [ ] sshd allows `jenkins` with pubkey auth
- [ ] `authorized_keys` 600, `.ssh` 700, owned by jenkins
- [ ] Disk ≥ 20 GB free; prune cron in place

**Jenkins UI**
- [ ] Credential `ssh-agent-<agent-name>` created
- [ ] Node created: workdir, labels, usage, known-hosts strategy
- [ ] Node log: `Agent successfully connected and online`

**Pipeline**
- [ ] `dockerhub` / `github` credentials added
- [ ] Smoke job shows `Running on <agent-name>`
- [ ] App Jenkinsfile builds and pushes from this node
