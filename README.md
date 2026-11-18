<div align="center">

# Homelab - Beyond the Certifications

3-node HA Kubernetes (K3s) cluster managed with Flux GitOps and Renovate-driven image automation

![K3s](https://img.shields.io/badge/K3s-HA%20etcd-FFC61C?logo=k3s&logoColor=black)
![Flux](https://img.shields.io/badge/GitOps-Flux-5468FF?logo=flux&logoColor=white)
![Fedora](https://img.shields.io/badge/OS-Fedora%2044-51A2DA?logo=fedora&logoColor=white)
![Proxmox](https://img.shields.io/badge/Hypervisor-Proxmox-E57000?logo=proxmox&logoColor=white)

[Architecture](#architecture-overview) · [Roadmap](#roadmap) · [Lessons Learned](#lessons-learned--troubleshooting)

</div>

## Highlights
- 3-node HA control plane with etcd quorum and a kube-vip floating API VIP
- GitOps with Flux: everything in the cluster is declared in this repo
- VLAN-isolated cluster network (802.1Q) with no path to the home LAN
- Mixed bare-metal and Proxmox-virtualized nodes
- Public services exposed through a Cloudflare Tunnel
- Monitoring with Prometheus + Grafana via Helm
- Automated image updates via Renovate

## Introduction
After obtaining my RHCSA and RHCE, I realized quickly that even though certifications are helpful, what matters the most is what you do beyond the certifications.

This repo contains all of the configuration and documentation of my homelab. 

My homelab is the place where I can try out and learn new things.

In addition, by self-hosting applications, it makes me feel responsible for the entire process of deploying and maintaining an application from A to Z. It forces me to think about backup strategies, security, scalability and the ease of deployment and maintenance.

## Architecture Overview

```mermaid
flowchart TB
    ISP((Internet))
    SW["TP-Link TL-SG108E
    (Switch)"]
    RTR["GL.iNet GL-A1300
    (Router)"]
    ISP --- RTR
    RTR --- SW

    subgraph K3S["K3s Cluster - VLAN 10"]
        VIP{{kube-vip}}
        CF[cloudflared]
        subgraph CTRL["Control plane / etcd - 3-node quorum"]
            T480["ThinkPad T480
            (Node 1)"]
            NUC["Intel NUC7CJYHN
            (Node 2)"]
            M920["ThinkCentre M920Q
            (Node 3)"]
        end
        subgraph SVC["Services"]
            LD[Linkding]
            ABS[AudioBookShelf]
            SITE[justintimejlew.com]
            MON[Prometheus/Grafana]
        end
        FLUX ---| Deploys | SVC
        FLUX --- CTRL
        CF -->|routes to ClusterIP| SVC
    end
    CFT((Cloudflare Tunnel)) --- CF
   
    SW --- T480
    SW --- NUC
    SW --- M920

    GH[GitHub repo] -->|HTTPS + PAT| FLUX[Flux]    
    VIP -.-|floats between nodes| CTRL
```

## Hardware
I decided to use a combination of an old laptop and old mini computers because they are affordable, reliable, and compact since I travel a lot for work. In addition, I wanted to a combination of bare-metal with a hypervisor-virtualized in my cluster.
- Lenovo ThinkPad T480 (i7-8650U/24GB/512GB) — Fedora Workstation 44, primary node
- Intel NUC7CJYHN (J4025/16GB/256GB) — Proxmox VE + Fedora Server 44, second node
- Lenovo ThinkCentre M920Q (i7-8700T/16GB/512GB) — Fedora Server 44, third node
- GL.iNet GL-A1300 — travel/VPN router
- TP-Link TL-SG108E — managed switch
- CyberPower ST425 — UPS

## Software Stack
- OS: Fedora
- Orchestration: K3s
- GitOps: Flux (HTTPS/PAT auth)
- Networking: Cloudflare Tunnel exposes services externally — no LoadBalancer or Ingress controller needed; `cloudflared` routes directly to each service's ClusterIP from inside the cluster.

### VLAN 1 Configuration - Personal
| Device | IP |
|---|:---:|
| TL-SG108E Switch | .5 |
| GL-A1300 (VLAN 1 Gateway) | .8 |
| T480 (node) | .51 |
| DHCP (personal devices) | .100-.200 |

### VLAN 10 Configuration - Cluster
| Device | IP |
|---|:---:|
| GL-A1300 (VLAN 10 Gateway) | .1 |
| Promox Host Management | .5 |
| K3s VIP (kube-vip) | .10 |
| T480 (node1, control) | .11 |
| NUC (node2, control) | .12 |
| M920Q (node3, control) | .13 |

## Deployed Services

| | Service | Purpose | Status |
|:---:|---|---|:---:|
| <img src="doc/img/linkding.svg" width="40" alt="Linkding"> | **Linkding** | Bookmark manager | ✅ [Live](https://linkding.justintimejlew.com/) |
| <img src="doc/img/audiobookshelf.svg" width="40" alt="AudioBookShelf"> | **Audiobookshelf** | Audiobook server | ✅ [Live](https://audiobookshelf.justintimejlew.com) |
| <img src="doc/img/site.svg" width="40" alt="Personal Website"> | **justintimejlew.com** | Personal website | ✅ [Live](https://justintimejlew.com) |
| <img src="doc/img/mealie.svg" width="40" alt="Mealie"> | **Mealie** | Recipe/meal planning | ⏳ Not live |

## Repo Structure

I decided to use Flux's recommended repository structure and below only shows the skeleton of the necessary levels...not all of my repository levels.
```
├── apps
│   ├── base
│   │   └── linkding
│   │       ├── deployment.yaml
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       └── storage.yaml
│   └── staging
│       └── linkding
│           └── kustomization.yaml
├── clusters
│   └── staging
│       ├── apps.yaml
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── infrastructure.yaml
├── infrastructure
│   ├── base
│   │   └── kube-vip
│   │       ├── daemonset.yaml
│   │       ├── kustomization.yaml
│   │       └── rbac.yaml
│   └── staging
│       └── kube-vip
│           └── kustomization.yaml
└── monitoring
```

## Roadmap

| Category | Task | Status | Date |
|---|---|:---:|:---:|
| Network | Segment network on 802.1Q VLAN 10 trunk for security isolation | ✅ | 09/16/26 |
| Homelab | Join M920Q as 3rd node for HA etcd quorum | ✅ | 09/27/26 |
| Homelab | Exposed Linkding to the internet | ✅ | 10/19/26 |
| Homelab | Set up Ingress with Traefik | ✅ | 10/23/26 |
| Homelab | Add monitoring stack via Helm | ✅ | 11/09/26 |
| Homelab | Automate Flux image updates | ✅ | 11/12/26 |
| Homelab | Self-hosted personal website | ✅ | 11/15/26 |
| Homelab | Exposed AudioBookShelf to the internet | ✅ | 11/16/26 |
| Homelab | Reconfigured Grafana into Cloudflare tunnel | ✅ | 11/18/26 |
| Cloud & IaC | Stand up Terraform-managed repo/Azure foundation | ⏳ | |
| Cloud & IaC | Learn IaC fundamentals and Terraform modules | ⏳ | |
| Cloud & IaC | Provision Azure Kubernetes Service (AKS) with Terraform | ⏳ | |
| Application & Database | Deploy n8n and expose it publicly | ⏳ | |
| Application & Database | Deploy Postgres and connect n8n to it | ⏳ | |
| Application & Database | Explore running databases in Kubernetes (CloudNativePG) | ⏳ | |
| Application & Database | Configure and test CNPG backup/restore workflows | ⏳ | |
| Production Readiness | Harden AKS cluster access and apply operational best practices | ⏳ | |
| Production Readiness | Deploy a production-grade application with production-grade monitoring | ⏳ | |
| Production Readiness | Handle staging environment fixes and production cluster/customer onboarding | ⏳ | |
| GitOps | Extend Flux GitOps patterns across staging and production environments | ⏳ | |

## Lessons Learned / Troubleshooting

<details>
<summary><strong>Flux silently failed on a typo: <code>app/v1</code> instead of <code>apps/v1</code></strong></summary>

A typo in the Linkding deployment manifest (`apiVersion: app/v1`) made Flux fail reconciliation with no clear error. I only found it by checking `flux get kustomizations`. `yamllint` did not flag it.

**Fix:** corrected the value to `apps/v1`.
</details>

<details>
<summary><strong>PVC volume definition errors blocked pod scheduling</strong></summary>

A malformed `persistentVolumeClaim` volume definition kept the Linkding pod from scheduling. The names of the volume blocks did not match, so Flux never reconciled or applied my changes.

**Fix:** aligned the volume names across the manifest sections.
</details>

<details>
<summary><strong>SSH auth failures across chezmoi, Flux, and DevPod</strong></summary>

SSH authentication failed consistently in my containerized and constrained environments.

**Fix:** switched to HTTPS + GitHub PAT everywhere.
</details>

<details>
<summary><strong>KUBECONFIG defaulting to a root-owned kubeconfig</strong></summary>

After I moved the cluster from Rancher on my iMac to K3s on my T480, `kubectl` commands failed with permission errors because they defaulted to `/etc/rancher/k3s/k3s.yaml` instead of `~/.kube/config`.

**Fix:** set `KUBECONFIG` in my `dot_zshrc`, which is managed by Chezmoi + Mise.
</details>

<details>
<summary><strong>SELinux blocking DevPod workspace mounts on Fedora</strong></summary>

DevPod worked when I ran it on Ubuntu. After I switched to Fedora, workspace mounts hit permission errors from SELinux.

**Fix:** used `semanage` and `restorecon` to set the `container_file_t` context instead of `user_file_t`, rather than running `setenforce 0`.
</details>

<details>
<summary><strong>VLAN segmentation surfaces silent firewall blocks</strong></summary>

Segmenting K3s nodes onto their own VLAN (isolated from the home network) exposed that Fedora's `firewalld` was blocking inter-node etcd traffic by default. The symptom was a 13+ minute hang on server join with `etcd: request timed out`, which is not an obvious firewall error.

**Fix:** explicitly open K3s's required ports on every server node:
- `6443/tcp` — API server
- `2379-2380/tcp` — etcd client/peer
- `8472/udp` — flannel VXLAN overlay
- `10250/tcp` — kubelet API
</details>

<details>
<summary><strong>Failed etcd joins leave stale state on the leader, not just the joining node</strong></summary>

A join attempt that fails partway (e.g. due to the firewall issue above) can register the joining node as an etcd learner on the *leader* before failing. Simply wiping the failed node (`k3s-uninstall.sh`) and retrying is not enough because the leader still holds a stale learner entry and rejects the retry with `duplicate node name found`.

**Fix:** on the leader, remove the stale member directly via `etcdctl member remove <id>`, and also check for/delete a leftover `<node>.node-password.k3s` secret in `kube-system` — K3s tracks node identity separately from etcd membership, and either one alone can cause a collision.
</details>

<details>
<summary><strong>kube-vip as a static pod is a single point of failure for HA</strong></summary>

I initially deployed kube-vip as a single static pod manifest on one control-plane node. This "worked" until that specific node went down, then the VIP went down with it, defeating the purpose of HA.

**Fix:** deploy kube-vip as a DaemonSet across all control-plane nodes (with RBAC for leader election), so any surviving node can claim the VIP via `vip_leaderelection`.
</details>

<details>
<summary><strong>A wiped node is not just a K3s problem — it can take Flux down too</strong></summary>

Resetting the control-plane node (`k3s-uninstall.sh`) during the VLAN migration also wiped Flux's CRDs, controllers, and git auth secret, since they were running as in-cluster resources. This was not obvious until `flux get kustomizations` failed with a missing-API-resource error days later.

**Fix:** since the bootstrap manifests (`gotk-components.yaml`, `gotk-sync.yaml`) were already committed to git, recovery was a straight `kubectl apply` of both — no new GitHub token needed unless the auth secret itself was also lost (it was, and had to be recreated with `flux create secret git`).
</details>

<details>
<summary><strong>Token rotation does not propagate automatically</strong></summary>

I wanted to learn how to rotate a token in the event that one needed to be changed.  Running `k3s token rotate` on the server only generates a new token, it does not update the token stored locally on any node (`/etc/systemd/system/k3s.service.env`).

**Fix:** manually update the token in each node's env file and restart k3s one node at a time, confirming quorum is maintained (2-of-3 minimum) before moving to the next.
</details>

<details>
<summary><strong>New app manifests were ignored because they weren't listed in <code>kustomization.yaml</code></strong></summary>

Flux only applies resources listed in `kustomization.yaml`. When I added `service.yaml` and `cloudflare.yaml` for the Cloudflare Tunnel, Flux skipped them with no error, so the app wasn't reachable.

**Fix:** added each new manifest to the `resources:` list, then committed, pushed, and ran `flux reconcile kustomization <name>`. Do this every time an app is added or updated.
</details>

<details>
<summary><strong>Ensure secret names are correct when creating them</strong></summary>

I accidentally copied the same <code>kubectl create secret</code> command using the <code>linkding-container-env</code> name, but the cronjob I created expected <code>renovate-container-env</code> name. The pod failed with a CreateContainerConfigError.

**Fix:** I was able to update the yaml file within my homelab and that allowed Flux to reconcile the changes correctly.
</details>

<details>
<summary><strong>Use one cloudflared deployment for all services on a tunnel</strong></summary>

I planned to add a second <code>cloudflared</code> deployment and a second <code>tunnel-credentials.yaml</code> for audiobookshelf, using the same tunnel as justintimejlew.com. Each <code>cloudflared</code> pod only knows the ingress rules in its own <code>config.yaml</code>. Cloudflare sends requests to any connector on the tunnel, so some requests would reach a pod with no rule for the hostname and return a 404.

**Fix:** I kept one <code>cloudflared</code> deployment in <code>apps/staging/justintimejlew</code> and added one ingress rule for each hostname in its ConfigMap. The rule for <code>audiobookshelf.justintimejlew.com</code> goes above the final <code>http_status:404</code> catch-all rule. I left <code>cloudflare.yaml</code> and <code>tunnel-credentials.yaml</code> commented out in the audiobookshelf kustomization.
</details>

<details>
<summary><strong>Use the full service name when the app is in another namespace</strong></summary>

My <code>cloudflared</code> pods run in the <code>justintimejlew</code> namespace and audiobookshelf runs in the <code>audiobookshelf</code> namespace. A short service name like <code>http://audiobookshelf:3005</code> only resolves inside the same namespace, so the tunnel could not find the service. I also considered adding <code>../audiobookshelf</code> to the justintimejlew kustomization. The <code>namespace</code> setting there would have moved the audiobookshelf resources into the wrong namespace.

**Fix:** I used the <code>&lt;service&gt;.&lt;namespace&gt;</code> format in the ingress rule: <code>http://audiobookshelf.audiobookshelf:3005</code>. I kept each app in its own kustomization. I also added a proxied CNAME for the subdomain that points to the tunnel, and I restarted <code>cloudflared</code> with <code>kubectl -n justintimejlew rollout restart deploy/cloudflared</code> because it does not reload its config when the ConfigMap changes.
</details>

<details>
<summary><strong>Encrypt the Grafana admin password with SOPS before exposing it to the internet</strong></summary>

I put the Grafana admin password in plain text as <code>adminPassword</code> in <code>release.yaml</code> and committed it to my public homelab repo. At that point Grafana was only reachable locally through a port forward, so it was low risk. Once I routed Grafana through the Cloudflare Tunnel, anyone could read that password in the repo and log in. The password also stays in the Git history after I remove it from the file.

**Fix:** I moved the password into a Kubernetes Secret and encrypted it with SOPS before I committed it. I created the Secret with <code>kubectl create secret generic grafana-admin --dry-run=client -o yaml</code> and encrypted it with <code>sops -e -i</code>. I used <code>read -rs</code> to enter the password so it never appeared on the screen or in my shell history. Then I pointed the Helm values at the Secret with <code>admin.existingSecret</code> and removed <code>adminPassword</code>. I confirmed that the values showed <code>ENC[AES256_GCM,...]</code> before I committed. I set a new password, because the old one is still in the Git history.
</details>
