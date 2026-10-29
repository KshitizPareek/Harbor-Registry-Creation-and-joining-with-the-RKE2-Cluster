# Harbor Registry on RHEL 9 and RKE2 Integration

Set up a private Harbor registry on RHEL 9 and configure an RKE2 cluster to pull all images through it.

## 1. Environment

| Role | IP | Notes |
| --- | --- | --- |
| Harbor registry | `3.80.35.144` | RHEL 9, Docker CE, Harbor v2.12.3 |
| RKE2 master | `98.89.28.73` | `rke2-server` |
| RKE2 worker | `3.80.32.15` | `rke2-agent` |

**Ports:** Harbor serves HTTPS on **443** and HTTP on 80. No port number is needed in image names or registry URLs (for example `3.80.35.144/library/hello-world`).

**Recommendation:** Attach an Elastic IP to the Harbor instance. If the public IP changes, the certificate and both nodes' configs must be redone.

## 2. Install Docker on the Harbor Host (RHEL 9)

Run as root on `3.80.35.144`.

```bash
sudo su -

# Update and install prerequisites
dnf -y update
dnf -y install dnf-plugins-core ca-certificates curl tar openssl nano

# Remove conflicting packages (RHEL ships podman and runc)
dnf -y remove docker docker-client docker-client-latest docker-common docker-latest \
  docker-latest-logrotate docker-logrotate docker-engine podman runc 2>/dev/null || true

# Add Docker's RHEL repository and install
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker and enable on boot
systemctl enable --now docker
docker version
```

If the `rhel` repository causes problems, use `https://download.docker.com/linux/centos/docker-ce.repo` instead.

## 3. Install Harbor

### 3.1 Download and extract

```bash
cd /root
curl -LO https://github.com/goharbor/harbor/releases/download/v2.12.3/harbor-offline-installer-v2.12.3.tgz
tar zxvf harbor-offline-installer-v2.12.3.tgz
cd harbor
```

### 3.2 Generate a self-signed certificate

The certificate must be issued for the Harbor IP, with a matching Subject Alternative Name.

```bash
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=3.80.35.144" \
  -addext "subjectAltName=IP:3.80.35.144"
```

Verify:

```bash
openssl x509 -in /root/harbor/tls.crt -noout -subject -ext subjectAltName
```

Expected output: `CN=3.80.35.144` and `IP Address:3.80.35.144`.

### 3.3 Configure `harbor.yml`

```bash
cp harbor.yml.tmpl harbor.yml
nano harbor.yml
```

Set the following values:

```yaml
hostname: 3.80.35.144

https:
  port: 443
  certificate: /root/harbor/tls.crt
  private_key: /root/harbor/tls.key
```

Also change `harbor_admin_password` from the default.

### 3.4 Open the firewall

```bash
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
```

In the AWS security group for the Harbor instance, allow inbound **443** from:

- `98.89.28.73/32` (master)
- `3.80.32.15/32` (worker)
- your own IP, for the UI and for pushing images

### 3.5 Install and verify

```bash
./install.sh
docker compose ps
curl --cacert /root/harbor/tls.crt https://3.80.35.144/api/v2.0/ping
```

All containers should become `healthy`, and the `curl` should return `Pong`.

To re-apply configuration changes later:

```bash
cd /root/harbor
docker compose down
./install.sh
```

Data under `/data` is preserved.

### 3.6 Trust the certificate in Docker (Harbor host)

The folder name has no port because Harbor uses 443.

```bash
mkdir -p /etc/docker/certs.d/3.80.35.144
cp /root/harbor/tls.crt /etc/docker/certs.d/3.80.35.144/ca.crt
```

### 3.7 Test push and pull

```bash
docker login 3.80.35.144
docker pull hello-world
docker tag hello-world 3.80.35.144/library/hello-world
docker push 3.80.35.144/library/hello-world
```

## 4. Configure Harbor for the Cluster

In the Harbor UI at `https://3.80.35.144`:

### 4.1 Add upstream registry endpoints

Go to **Administration → Registries → New Endpoint** and add:

| Name | Provider | Endpoint URL |
| --- | --- | --- |
| dockerhub | Docker Hub | `https://registry-1.docker.io` |
| k8s | Docker Registry | `https://registry.k8s.io` |
| quay | Quay | `https://quay.io` |

### 4.2 Create proxy-cache projects

Go to **Projects → New Project**, tick **Proxy Cache**, and select the matching endpoint:

| Project | Endpoint |
| --- | --- |
| `dockerhub-proxy` | dockerhub |
| `k8s-proxy` | k8s |
| `quay-proxy` | quay |

### 4.3 Create a pull-only robot account

Go to **Administration → Robot Accounts → New Robot Account**:

- Type: system robot
- Permissions: **pull** on `dockerhub-proxy`, `k8s-proxy` and `quay-proxy`
- Save the full username (for example `robot$r1`) and the token. The token is shown only once.

Keep the cluster robot pull-only. Pushes should come only from a separate account used on the build machine.

## 5. Configure the RKE2 Nodes

Repeat these steps on **both** the master (`98.89.28.73`) and the worker (`3.80.32.15`).

### 5.1 Place the Harbor certificate

SSH to the node and create the file. Paste the output of `cat /root/harbor/tls.crt` from the Harbor host:

```bash
sudo mkdir -p /etc/rancher/rke2
sudo nano /etc/rancher/rke2/harbor-ca.crt
```

Verify:

```bash
openssl x509 -in /etc/rancher/rke2/harbor-ca.crt -noout -subject -ext subjectAltName
```

AWS usually disables root SSH, so copying with `scp` as root from the Harbor host fails with `Permission denied (publickey)`. Pasting the certificate manually avoids this.

### 5.2 Create `registries.yaml`

```bash
sudo tee /etc/rancher/rke2/registries.yaml > /dev/null <<'EOF'
mirrors:
  docker.io:
    endpoint:
      - "https://3.80.35.144"
    rewrite:
      "^(.*)$": "dockerhub-proxy/$1"
  registry.k8s.io:
    endpoint:
      - "https://3.80.35.144"
    rewrite:
      "^(.*)$": "k8s-proxy/$1"
  quay.io:
    endpoint:
      - "https://3.80.35.144"
    rewrite:
      "^(.*)$": "quay-proxy/$1"

configs:
  "3.80.35.144":
    auth:
      username: "robot$r1"
      password: "<ROBOT_TOKEN>"
    tls:
      ca_file: /etc/rancher/rke2/harbor-ca.crt
EOF
```

Replace `<ROBOT_TOKEN>` with the robot's secret. Use the quoted `'EOF'` so the shell doesn't expand the `$` in the username.

### 5.3 Restart RKE2

Restart the master first and wait for it to come back before restarting the worker.

```bash
# Master (98.89.28.73)
sudo systemctl restart rke2-server
sudo systemctl status rke2-server

# Worker (3.80.32.15)
sudo systemctl restart rke2-agent
```

## 6. Verify

On either node:

```bash
# Test a pull through the mirror
sudo /var/lib/rancher/rke2/bin/crictl \
  --config /var/lib/rancher/rke2/agent/etc/crictl.yaml \
  pull docker.io/library/hello-world

# Check the cluster
sudo /var/lib/rancher/rke2/bin/kubectl \
  --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

After a successful pull, the image appears in the `dockerhub-proxy` project in Harbor. The generated containerd mirror config can be checked under `/var/lib/rancher/rke2/agent/etc/containerd/`.

## 7. Enforce Harbor-Only Images

Mirrors redirect pulls, but they do not stop a manifest from naming another registry. Add these controls as well.

### 7.1 Network egress

In the security groups for the master and worker, allow outbound HTTPS only to `3.80.35.144/32` and to what the cluster needs (DNS, the AWS API, and traffic between the nodes). With general internet access blocked, nothing can be pulled from other registries.

### 7.2 Admission policy (Kyverno example)

Install Kyverno, then apply a policy that allows only Harbor images in user namespaces:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: harbor-only
      match:
        any:
          - resources:
              kinds:
                - Pod
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - kyverno
      validate:
        message: "Images must come from 3.80.35.144."
        pattern:
          spec:
            =(initContainers):
              - image: "3.80.35.144/*"
            containers:
              - image: "3.80.35.144/*"
```

System namespaces are excluded because RKE2's own images are referenced by their upstream names and reach the node through the mirrors. With this policy, workloads in other namespaces must reference images as `3.80.35.144/<project>/<image>` (for example `3.80.35.144/dockerhub-proxy/library/nginx:1.27`).

### 7.3 Push control

The cluster robot has pull permission only, so images can be pushed only from machines where someone logs in with a push-capable account.

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `scp: Permission denied (publickey)` | Root SSH is disabled on AWS | Paste the cert manually, or use `ec2-user` with the `.pem` key and `/tmp` plus `sudo mv` |
| `scp: stat local ...: No such file` | Command run on the wrong machine | Run it on the Harbor host, or paste the cert manually |
| `x509: certificate signed by unknown authority` | Cert on the node is wrong or incomplete | Re-paste `tls.crt` and verify with `openssl x509` |
| `x509: cannot validate certificate for <ip>` | Cert was issued for a different IP | Regenerate the cert with the correct IP and SAN, then re-run `./install.sh` |
| `401 Unauthorized` | Wrong robot credentials, or no access to the project | Check the username and token, and the robot's project permissions |
| `404` or `project not found` | Project names don't match the `rewrite` rules | Create projects named exactly `dockerhub-proxy`, `k8s-proxy`, `quay-proxy` |
| `connection refused` or timeout on 443 | Security group or firewalld blocking | Allow inbound 443 from the node IPs and run `firewall-cmd --list-ports` |
| `connection refused` on 8080 | Harbor does not listen on 8080 | Use 443 with no port in URLs |
| `docker-compose: command not found` during install | Only the Compose plugin is installed | Create a `docker-compose` wrapper script that runs `docker compose "$@"` |
| Permission errors on `/data` | SELinux denial | Check `ausearch -m avc -ts recent` before considering `setenforce 0` |

## 9. Maintenance Notes

- **Certificate expiry:** the self-signed cert is valid for 365 days. Regenerate it before expiry, re-run `./install.sh`, and update `harbor-ca.crt` on both nodes, then restart RKE2.
- **IP changes:** if the Harbor IP changes, regenerate the cert for the new IP and update `harbor.yml`, the Docker trust folder, and `registries.yaml` and `harbor-ca.crt` on both nodes.
- **Robot token rotation:** refresh the secret in Harbor (Administration → Robot Accounts) and update `registries.yaml` on both nodes, then restart RKE2. Rotate the token immediately if it has been shared in a chat, ticket or log.
- **Admin password:** change the default Harbor admin password after first login.
- **Longer term:** replace the self-signed cert with one issued for a DNS name (for example `harbor.example.com`) so nodes don't need a custom CA file.
