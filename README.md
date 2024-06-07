# minilab

Notes, setup steps, ansible playbooks, and IaC for my minilab

## Hardware

- 1x HP 800 G6 with 10g nic will run Truenas
- 1x HP 800 G9 with 10g nic will run Proxmox
- 5x HP 800 G6 with 2.5g nic will host k8s lab cluster
- 2x HP 800 G6 with 2.5g nic reserved - locked out of vpro/bios

Some network equipment

- 1x Unifi Dream Machine
- 1x [Cheap 2.5gb switch](https://www.amazon.com/dp/B0CT2F3ZDM) with 2x 10gb ports

### Proxmox

Several services are required to bootstrap and run a talos k8s cluster: MeshCentral, Sidero Omni, DNS, Auth, Netboot/tftp, Vault (perhaps?).

### k8s

These 5 nodes will be deployed with Talos Linux. All nodes are workers and masters. ArgoCD and stateless apps are the eventual goal.

### truenas

Single node with 10gbe. Dual 2TB SSDs mirrored for cluster/app storage provided by Minio. 8TB ssd for file storage over nfs/cifs.

## Network Design

- Machines' vpro enabled ports are connected the 1g ports on the UDMP where the vlan 2 subnet will have NO internet access, but will be able to connect to a MeshCentral server on the prod vlan for management.
- The 10gb port on UDMP has a 2.5gb transeiver connected to a 2.5gb port on the cheap switch as an uplink. The uplink port is marked vlan 3 for prod network.
- 2x 10gb machines connect to the cheap 10gb ports, the rest to thet 2.5gb ports.

| Subnet          | vlan | Purpose                            | Switch      |
| --------------- | ---- | ---------------------------------- | ----------- |
| 192.168.70.0/24 | 2    | 1gb vpro ports on the mini pc      | UDMP        |
| 192.168.80.0/24 | 3    | 2.5/10gb ports - data/prod network | Cheap 2.5gb |

Service Domain: bauer.mt
Management Domain mgmt.bauer.mt
vPro Domain: amt.bauer.mt

| Device Group | Hostname | Multigig IP   | 1gb admin IP  | AMT IP        | CPU       | Memory   |
| ------------ | -------- | ------------- | ------------- | ------------- | --------- | -------- |
| nas          | truenas  | 192.168.80.10 | 192.168.70.10 | 192.168.70.50 | i5-10500  | 32g ddr4 |
| proxmox      | pve      | 192.168.80.11 | 192.168.70.11 | 192.168.70.51 | i5-13500T | 48g ddr5 |
| talos        | lab-00   | 192.168.80.15 | 192.168.70.15 | 192.168.70.55 | i7-10700  | 48g ddr4 |
| talos        | lab-01   | 192.168.80.16 | 192.168.70.16 | 192.168.70.56 | i5-10500T | 48g ddr4 |
| talos        | lab-02   | 192.168.80.17 | 192.168.70.17 | 192.168.70.57 | i5-10500T | 48g ddr4 |
| talos        | lab-03   | 192.168.80.18 | 192.168.70.18 | 192.168.70.58 | i5-10500T | 16g ddr4 |
| talos        | lab-04   | 192.168.80.19 | 192.168.70.19 | 192.168.70.59 | i5-10500T | 16g ddr4 |

| Service  | VM/CT | Hostname | Primary       | Secondary IP | Cores | Memory | Disk | Note                           |
| -------- | ----- | -------- | ------------- | ------------ | ----- | ------ | ---- | ------------------------------ |
| DNS      | CT    | pihole   | 192.168.80.3  | -            | 2     | 1G     | 10G  |                                |
| Plex     | CT    | plex     | 192.168.80.81 | -            | 10    | 16G    | 60G  |                                |
| Jellyfin | CT    | jellyfin | 192.168.80.82 | -            | 10    | 16G    | 60G  |                                |
| Docker   | VM    | omni     | 192.168.80.80 | -            | 4     | 8G     | 100G | Sidero Omni, Auth, MeshCentral |

## Install /configure Proxmox

1. disable enterprise repo and enable community repository
1. add nfs iso storage from truenas
1. `pveam update` to make LXC templates available
1. create containers manually from debian template (ansible someday maybe)
1. [create vm templates](https://pycvala.de/blog/proxmox/create-your-own-debian-12-cloud-init-template/) for Debian 12 / backports

## Debian VM setup

1. Clone template in proxmox and modify hardware config as needed.
1. configure cloud-init for each node with a user 'ansible' and ssh keys for login
1. Boot vm and use ansible to files system on disk (if enlarged) and update packages.

## Create VM from template for docker services

1. Create VM from template, modify hard drive size, ssd, and resources.
1. boot vm and use ansible to install qemu agent, resize filesystem to match drive:

```
ansible-galaxy install -r ./ansible/requirements.yml
ansible-playbook ./ansible/vm/cloud-img-tasks.yml -i ./ansible/hosts.ini
ansible-playbook ./ansible/vm/qemu-agent.yml -i ./ansible/hosts.ini
ansible-playbook ./ansible/vm/users.yml -i ./ansible/hosts.ini
ansible-playbook ./ansible/vm/docker.yml -i ./ansible/hosts.ini
```

# TODO ^ Make those proper roles

## Install minio app on truenas

1. [official guide from truenas](https://www.truenas.com/docs/scale/scaletutorials/apps/communityapps/minioapp/)

## Enable kubectl for truenas apps (k3s) (for troubleshooting only, if desired)

1. an app must be installed and running or connection to k3s won't be possible. (install minio as above)
1. Use [heavyscript](https://github.com/Heavybullets8/heavy_script?tab=readme-ov-file#the-menu) with the `--api` flag to enable access remotely with kubectl
1. copy kubectl form `/etc/rancher/k3s/k3s.yaml` from truenas to local machine (cat file, copy paste)
1. modify the server address and context details.
1. combine with existing kubeconfig if necessary.

```
export KUBECONFIG=/Users/brian/.kube/individual-configs/<configname1>:/Users/brian/.kube/individual-configs/<confignameN>
kubectl config view --flatten > one-config.yaml
mv one-config ~/.kube/config
```

## Talos linux

1. Install Sidero Omni for network booting and lifecycle management of Talos k8s deployments
1. do a lot of reading: https://a-cup-of.coffee/blog/talos/#introduction

## Configure traefik and automate certificate management.

1. Follow [this guide](https://technotim.live/posts/kube-traefik-cert-manager-le/). yaml files have been copied to this repository in `./manifests/prototype/...`
1. install traefik with the values file at `./manifests/prototype/traefik/values.yaml` `helm install --namespace=traefik traefik traefik/traefik --values=./manifests/prototype/traefik/values.yaml`
1. Set default certificate `kubectl apply -f manifests/prototype/traefik/dashboard/default-certificate-config.yaml`
1. todo ... this again

## DNS

PiHole for now, running in a container. This was a good guide for k8s: I'm modifying [this](https://github.com/JamesTurland/JimsGarage/tree/main/Kubernetes/Traefik-PiHole/Manifest/PiHole) starter code after watching [these video instructions](https://www.youtube.com/watch?v=2-IkRF8HyCc)

1. `kubectl apply -f ./manifests/prototype/pihole`

## MeshCentral

1. `kubectl apply -f ./manifests/prototype/meshcentral`
1. Manually modify the meshcentral `config.json` as described [here](https://github.com/Ylianst/MeshCentral/issues/5826) or [here](https://github.com/Ylianst/MeshCentral/issues/5826) unless the docker image has been updated with the requested env variable.

## netboot-xyz

1. `kubectl apply -f ./manifests/prototype/netboot-xyz`
