# MIS Midterm Part 1 - K3s on four EC2 instances

Full Name: **YOUR FULL NAME**

This runbook follows the four numbered requirements in the assignment and gives you a clean screenshot order for Notion.

## Important naming rule

Keep the assignment wording in the AWS **Name tag**, but do not put underscores in the Linux/K3s hostname.

| Assignment / EC2 Name tag | Linux and K3s hostname |
|---|---|
| `S1_Master` | `s1-master` |
| `S2_worker1` | `s2-worker1` |
| `S2_worker2` | `s2-worker2` |
| `Nginx_Proxy` | `nginx-proxy` |

K3s requires every node to have a unique valid hostname. The website itself will still display the assignment names such as `S1_Master` and `S2_Worker1`.

## Architecture

```text
Browser
  |
  | HTTP port 80
  v
Nginx_Proxy (public entry point)
  |-- /master/  --> S1_Master private IP:30081 --> master-site pod on s1-master
  |-- /worker1/ --> S2_worker1 private IP:30082 --> worker1-site pod on s2-worker1
  `-- /worker2/ --> S2_worker2 private IP:30083 --> worker2-site pod on s2-worker2
```

The three site pods are deliberately pinned to their matching K3s nodes. The Nginx proxy reaches them through Kubernetes NodePort Services.

## 1. Record your values first

Fill this table before running commands. Use private IPs for communication between EC2 instances.

| Instance | Public IPv4 | Private IPv4 | OS hostname |
|---|---|---|---|
| S1_Master |  |  | s1-master |
| S2_worker1 |  |  | s2-worker1 |
| S2_worker2 |  |  | s2-worker2 |
| Nginx_Proxy |  |  | nginx-proxy |

Also record:

- AWS region: `____________`
- Key pair path: `____________`
- Your current public IP: `____________/32`

## 2. Create the four EC2 instances

Use the same region, VPC, and subnet for all four instances.

Recommended classroom setup:

- AMI: Ubuntu Server 24.04 LTS, x86_64
- K3s nodes: `t3.small` or the instance type required by your instructor
- Nginx proxy: `t3.micro` is enough for this demo
- Storage: the default 8 GB is normally enough
- Auto-assign public IPv4: enabled, because you need SSH and browser screenshots
- Key pair: use the same key pair for all four instances

Create these EC2 Name tags exactly:

1. `S1_Master`
2. `S2_worker1`
3. `S2_worker2`
4. `Nginx_Proxy`

Take one screenshot of the EC2 Instances table showing all four names, the `Running` state, and the public/private IP columns.

## 3. Configure security groups

Create two security groups.

### `mis-k3s-sg`

Attach this security group to `S1_Master`, `S2_worker1`, and `S2_worker2`.

| Type | Protocol | Port | Source | Purpose |
|---|---:|---:|---|---|
| SSH | TCP | 22 | My IP | Administration |
| Custom TCP | TCP | 6443 | `mis-k3s-sg` | Workers join the K3s API |
| Custom UDP | UDP | 8472 | `mis-k3s-sg` | Flannel VXLAN between K3s nodes |
| Custom TCP | TCP | 10250 | `mis-k3s-sg` | Kubelet metrics/API between nodes |
| Custom TCP | TCP | 30081-30083 | `mis-proxy-sg` | Nginx reaches the three NodePorts |

Optional for direct browser screenshots of options 1-3:

| Type | Protocol | Port | Source |
|---|---:|---:|---|
| Custom TCP | TCP | 30081-30083 | My IP |

Remove the optional public NodePort rule after taking the screenshots.

### `mis-proxy-sg`

Attach this security group only to `Nginx_Proxy`.

| Type | Protocol | Port | Source | Purpose |
|---|---:|---:|---|---|
| SSH | TCP | 22 | My IP | Administration |
| HTTP | TCP | 80 | `0.0.0.0/0` | Public website access |

Keep the default outbound rule that allows all traffic. Do not expose UDP 8472 to the internet.

## 4. Set a persistent hostname inside every instance

SSH to each instance and run the same commands, changing only `<HOSTNAME>` to the correct value from the table below.

```bash
echo 'preserve_hostname: true' | sudo tee /etc/cloud/cloud.cfg.d/99-preserve-hostname.cfg
sudo hostnamectl set-hostname <HOSTNAME>
sudo reboot
```

Use one of these values per instance:

```text
S1_Master   -> s1-master
S2_worker1  -> s2-worker1
S2_worker2  -> s2-worker2
Nginx_Proxy -> nginx-proxy
```

The SSH connection will close during the reboot. Reconnect and verify:

```bash
hostnamectl --static
hostname
```

Expected values are `s1-master`, `s2-worker1`, `s2-worker2`, and `nginx-proxy`. Capture the verification output for each assignment section.

## 5. Option 1 - install the K3s server on S1_Master

SSH to `S1_Master` and run:

```bash
sudo apt update
sudo apt install -y curl
sudo ufw disable

curl -sfL https://get.k3s.io | sudo sh -s - server \
  --node-name s1-master \
  --write-kubeconfig-mode 644
```

Verify the server:

```bash
sudo systemctl is-active k3s
sudo kubectl get nodes -o wide
```

Both checks should show `active` / `Ready`.

Get the worker join token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy the token somewhere temporary. **Do not put the token in Notion and do not include it in a screenshot.** Treat it like a password.

## 6. Option 2 - join S2_worker1

SSH to `S2_worker1`. Replace `<MASTER_PRIVATE_IP>` and `<TOKEN>` with real values; do not type the angle brackets.

```bash
sudo apt update
sudo apt install -y curl
sudo ufw disable

curl -sfL https://get.k3s.io | sudo env \
  K3S_URL="https://<MASTER_PRIVATE_IP>:6443" \
  K3S_TOKEN="<TOKEN>" \
  K3S_NODE_NAME="s2-worker1" sh -
```

Verify on worker 1:

```bash
sudo systemctl is-active k3s-agent
hostnamectl --static
```

## 7. Option 3 - join S2_worker2

SSH to `S2_worker2` and run the same process with its own node name:

```bash
sudo apt update
sudo apt install -y curl
sudo ufw disable

curl -sfL https://get.k3s.io | sudo env \
  K3S_URL="https://<MASTER_PRIVATE_IP>:6443" \
  K3S_TOKEN="<TOKEN>" \
  K3S_NODE_NAME="s2-worker2" sh -
```

Verify on worker 2:

```bash
sudo systemctl is-active k3s-agent
hostnamectl --static
```

Return to `S1_Master` and verify the full cluster:

```bash
sudo kubectl wait --for=condition=Ready nodes --all --timeout=180s
sudo kubectl get nodes -o wide
```

Expected node names:

```text
s1-master
s2-worker1
s2-worker2
```

Take a screenshot that clearly shows all three nodes as `Ready`.

## 8. Deploy one website on each K3s node

The prepared manifest is `infrastructure/kubernetes/mis-websites.yaml`. It contains:

- A blue `S1_Master` site pinned to `s1-master`, exposed on NodePort `30081`
- A green `S2_Worker1` site pinned to `s2-worker1`, exposed on NodePort `30082`
- An orange `S2_Worker2` site pinned to `s2-worker2`, exposed on NodePort `30083`

### Personalize the file locally

Open `infrastructure/kubernetes/mis-websites.yaml` and replace all three occurrences of:

```text
YOUR FULL NAME
```

with your real full name.

### Upload from Windows PowerShell

Replace the key path and master public IP:

```powershell
scp -i "C:\path\to\your-key.pem" "C:\Users\Admin\Desktop\Codebases\Random\PPUA\infrastructure\kubernetes\mis-websites.yaml" ubuntu@MASTER_PUBLIC_IP:/home/ubuntu/
```

If you are using EC2 Instance Connect instead, create `/home/ubuntu/mis-websites.yaml` with `nano` and paste the entire prepared manifest into it.

### Apply on S1_Master

```bash
sudo kubectl apply -f /home/ubuntu/mis-websites.yaml

sudo kubectl rollout status deployment/master-site --timeout=180s
sudo kubectl rollout status deployment/worker1-site --timeout=180s
sudo kubectl rollout status deployment/worker2-site --timeout=180s

sudo kubectl get pods -o wide
sudo kubectl get services
```

The `NODE` column must show each pod on its matching node. This is important evidence that the three sites are actually running through Kubernetes on the requested instances.

### Test options 1-3 directly

If you added the optional My IP security-group rule for NodePorts, open:

```text
http://MASTER_PUBLIC_IP:30081
http://WORKER1_PUBLIC_IP:30082
http://WORKER2_PUBLIC_IP:30083
```

Capture each page in its matching Notion section. Each page should show the Kubernetes message, the instance name, and your full name.

You can also test all services from `S1_Master`:

```bash
curl http://127.0.0.1:30081
curl http://127.0.0.1:30082
curl http://127.0.0.1:30083
```

## 9. Option 4 - configure the Nginx proxy server

The prepared `infrastructure/nginx/mis-proxy.conf` file has three placeholders:

```text
MASTER_PRIVATE_IP
WORKER1_PRIVATE_IP
WORKER2_PRIVATE_IP
```

Replace them locally with the three K3s instances' private IPv4 addresses.

Upload the config from Windows PowerShell:

```powershell
scp -i "C:\path\to\your-key.pem" "C:\Users\Admin\Desktop\Codebases\Random\PPUA\infrastructure\nginx\mis-proxy.conf" ubuntu@PROXY_PUBLIC_IP:/home/ubuntu/
```

SSH to `Nginx_Proxy`, confirm its hostname, and install Nginx:

```bash
hostnamectl --static
sudo apt update
sudo apt install -y nginx curl
```

Before changing Nginx, confirm that the proxy instance can reach all three backends:

```bash
curl -I http://MASTER_PRIVATE_IP:30081/
curl -I http://WORKER1_PRIVATE_IP:30082/
curl -I http://WORKER2_PRIVATE_IP:30083/
```

Every command should return an HTTP response such as `HTTP/1.1 200 OK`. Then enable the proxy configuration:

```bash
sudo cp /home/ubuntu/mis-proxy.conf /etc/nginx/sites-available/mis-proxy
sudo ln -sf /etc/nginx/sites-available/mis-proxy /etc/nginx/sites-enabled/mis-proxy
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl enable --now nginx
sudo systemctl restart nginx
sudo systemctl is-active nginx
```

Expected results include `syntax is ok`, `test is successful`, and `active`.

Open these URLs in your browser:

```text
http://PROXY_PUBLIC_IP/master/
http://PROXY_PUBLIC_IP/worker1/
http://PROXY_PUBLIC_IP/worker2/
```

Opening `http://PROXY_PUBLIC_IP/` redirects to `/master/`.

## 10. Recommended Notion screenshot structure

Create one Notion page titled:

```text
MIS Midterm Part 1 - Kubernetes with K3s
Full Name: YOUR FULL NAME
```

### Environment overview

- Screenshot: all four EC2 instances running, with names and IP columns
- Screenshot: both security groups and their inbound rules
- Small table: instance Name tag, OS hostname, private IP, public IP

### Option 1 - S1_Master

- Screenshot: `hostnamectl --static` showing `s1-master`
- Screenshot: `systemctl is-active k3s` and the master node as `Ready`
- Screenshot: `kubectl get pods -o wide` showing `master-site` on `s1-master`
- Screenshot: browser page on port `30081`

### Option 2 - S2_worker1

- Screenshot: `hostnamectl --static` showing `s2-worker1`
- Screenshot: `systemctl is-active k3s-agent`
- Screenshot: `kubectl get pods -o wide` showing `worker1-site` on `s2-worker1`
- Screenshot: browser page on port `30082`

### Option 3 - S2_worker2

- Screenshot: `hostnamectl --static` showing `s2-worker2`
- Screenshot: `systemctl is-active k3s-agent`
- Screenshot: `kubectl get pods -o wide` showing `worker2-site` on `s2-worker2`
- Screenshot: browser page on port `30083`

### Option 4 - Nginx proxy

- Screenshot: `hostnamectl --static` showing `nginx-proxy`
- Screenshot: the three `proxy_pass` entries in `/etc/nginx/sites-available/mis-proxy`
- Screenshot: successful `sudo nginx -t` and active Nginx service
- Screenshots: `/master/`, `/worker1/`, and `/worker2/` through the proxy public IP

### Final proof

Use this compact command on `S1_Master` for a strong final screenshot:

```bash
echo '=== NODES ==='
sudo kubectl get nodes -o wide
echo '=== POD PLACEMENT ==='
sudo kubectl get pods -o wide
echo '=== SERVICES ==='
sudo kubectl get services
```

Before uploading screenshots, check that none contain the K3s token, private key contents, or unrelated personal information.

## Troubleshooting

### Worker does not join or stays NotReady

On the worker:

```bash
sudo journalctl -u k3s-agent -n 80 --no-pager
```

Check that:

- `K3S_URL` uses the master's **private** IP and port `6443`
- The token was copied exactly
- TCP `6443`, UDP `8472`, and TCP `10250` have the security-group rules shown above
- Every hostname/node name is unique

### A pod stays Pending

```bash
sudo kubectl get nodes
sudo kubectl describe pod POD_NAME
```

The node names must exactly match `s1-master`, `s2-worker1`, and `s2-worker2`, because the manifest uses those names in `nodeSelector`.

### You changed the full name but the old page remains

Reapply and restart the deployments:

```bash
sudo kubectl apply -f /home/ubuntu/mis-websites.yaml
sudo kubectl rollout restart deployment/master-site deployment/worker1-site deployment/worker2-site
sudo kubectl rollout status deployment/master-site
sudo kubectl rollout status deployment/worker1-site
sudo kubectl rollout status deployment/worker2-site
```

### Nginx shows 502 Bad Gateway

From `Nginx_Proxy`, run the three backend `curl -I` commands again. A failure means the private IP, NodePort, pod/service, or security-group source rule is wrong. Also verify:

```bash
sudo nginx -t
sudo journalctl -u nginx -n 80 --no-pager
```

### Direct browser URL times out

Confirm that the temporary `30081-30083` inbound rule uses **My IP** in `mis-k3s-sg`. The final proxy URLs need only public TCP port `80` on `mis-proxy-sg`.

## Finish safely

- Remove the temporary public NodePort rule after screenshots.
- Never publish the K3s join token or `.pem` key.
- Stop or terminate the EC2 instances after submission so they do not continue generating AWS charges.

## Official references

- K3s quick start: https://docs.k3s.io/quick-start
- K3s networking requirements: https://docs.k3s.io/installation/requirements
- Kubernetes NodePort Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes node selection: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/
- AWS security-group rules: https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html
- Nginx `proxy_pass`: https://nginx.org/en/docs/http/ngx_http_proxy_module.html
