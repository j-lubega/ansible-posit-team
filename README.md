# Ansible: Posit Team (Workbench + Connect)

Ansible playbooks that install **Posit Workbench** and **Posit Connect** on Ubuntu 24.04,
plus a **Molecule** scenario that tests the full deployment on throwaway Proxmox VMs.

The same playbooks deploy to any Ubuntu 24.04 VM: Proxmox, Azure, AWS, GCP or bare metal.
Only the way you create the VMs and build the inventory changes.

| Server    | Playbook              | Installs                                                                 | Port |
|-----------|-----------------------|--------------------------------------------------------------------------|------|
| Workbench | `posit-workbench.yml` | R, Python (uv), Jupyter, Quarto, Posit Workbench, Posit Pro ODBC drivers | 8787 |
| Connect   | `posit-connect.yml`   | R, Python (uv), Quarto, Posit Connect, Posit Pro ODBC drivers            | 3939 |

`site.yml` runs both.

References: [Posit docs](https://docs.posit.co/) ·
[Workbench install](https://docs.posit.co/ide/server-pro/admin/getting_started/installation/installation.html) ·
[Connect install](https://docs.posit.co/connect/admin/getting-started/local-install/server-install/) ·
[Install R](https://docs.posit.co/resources/install-r/) ·
[Install Python (uv)](https://docs.posit.co/resources/install-python-uv.html) ·
[Install Quarto](https://docs.posit.co/resources/install-quarto.html)

---

## Contents

1. [Repository layout](#1-repository-layout)
2. [Requirements](#2-requirements)
3. [Secrets](#3-secrets)
4. [Deploy to any existing VMs (quick start)](#4-deploy-to-any-existing-vms-quick-start)
5. [Proxmox](#5-proxmox)
6. [Azure](#6-azure)
7. [AWS](#7-aws)
8. [GCP](#8-gcp)
9. [Test with Molecule (Proxmox)](#9-test-with-molecule-proxmox)
10. [After deployment](#10-after-deployment)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Repository layout

```
.
├── site.yml                     # runs Workbench + Connect
├── posit-workbench.yml          # hosts: workbench
├── posit-connect.yml            # hosts: connect
├── inventory.example.ini        # copy to inventory.ini (git-ignored)
├── posit-poc.env.example        # copy to ~/posit-poc.env (never commit)
└── molecule/
    └── default/
        ├── molecule.yml         # two Proxmox VMs: posit-workbench, posit-connect
        ├── create.yml           # linked-clones VMs from a cloud-init template
        ├── destroy.yml          # stops and deletes the VMs
        ├── converge.yml         # imports ../../site.yml
        ├── verify.yml           # health checks
        └── requirements.yml     # Ansible collections
```

## 2. Requirements

**Target servers (one per product; Posit doesn't support installing several products on one server)**

| | Minimum for a POC | Notes |
|---|---|---|
| OS | Ubuntu 24.04 LTS, x86_64 | The playbooks assert this |
| CPU / RAM | 4 vCPU / 8 GB each | The playbooks warn if smaller |
| Disk | 60 GB+ | R/Python packages and content grow over time |
| Network | Outbound HTTPS | `cdn.posit.co`, `dl.posit.co`, `packagemanager.posit.co`, `github.com`, `astral.sh` |
| Access | SSH user with passwordless `sudo` | |
| Licenses | Workbench and Connect license keys | Trial keys work |

**Control node** (laptop, CI agent, or a small VM such as `ansible-ctl`)

```bash
sudo apt install -y git python3-venv
python3 -m venv ~/.venvs/molecule && source ~/.venvs/molecule/bin/activate
pip install ansible-core molecule proxmoxer requests
ansible-galaxy collection install -r molecule/default/requirements.yml
```

## 3. Secrets

Nothing sensitive is stored in this repo. Secrets come from environment variables or `-e` at run time.

```bash
cp posit-poc.env.example ~/posit-poc.env   # outside the repo
chmod 600 ~/posit-poc.env
nano ~/posit-poc.env
source ~/posit-poc.env
```

| Variable | Used for |
|---|---|
| `PWB_LICENSE` | Workbench license key |
| `PCT_LICENSE` | Connect license key |
| `PVE_HOST`, `PVE_NODE`, `PVE_USER`, `PVE_TOKEN_ID`, `PVE_TOKEN_SECRET` | Proxmox API (Molecule only) |

Both playbooks use `vars_prompt`: they ask for license keys interactively unless you pass them
with `-e`. Pass them with `-e` in automation:

```bash
-e "workbench_license_key=$PWB_LICENSE connect_license_key=$PCT_LICENSE workbench_test_password=<password>"
```

For team use, consider `ansible-vault` or your cloud's secret store (Azure Key Vault, AWS Secrets
Manager, GCP Secret Manager) and your CI system's secret variables.

## 4. Deploy to any existing VMs (quick start)

Every environment below ends with this step.

```bash
cp inventory.example.ini inventory.ini
nano inventory.ini            # set the two IPs, SSH user and key
ansible -i inventory.ini all -m ping
ansible-playbook -i inventory.ini site.yml \
  -e "workbench_license_key=$PWB_LICENSE connect_license_key=$PCT_LICENSE workbench_test_password=<password>"
```

Deploy one product only:

```bash
ansible-playbook -i inventory.ini posit-workbench.yml -e "workbench_license_key=$PWB_LICENSE"
ansible-playbook -i inventory.ini posit-connect.yml   -e "connect_license_key=$PCT_LICENSE"
```

Pin versions (recommended outside a POC) by overriding the playbook variables, e.g.
`-e workbench_version=<version> -e connect_version=<version> -e quarto_version=<version>`.

---

## 5. Proxmox

### 5.1 One-time setup (Proxmox host shell)

**Cloud-init template (VM 9000)**

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
qm create 9000 --name ubuntu2404-tmpl --memory 2048 --cores 2 --cpu host \
  --net0 virtio,bridge=vmbr0 --scsihw virtio-scsi-pci --agent 1
qm importdisk 9000 noble-server-cloudimg-amd64.img local-lvm
qm set 9000 --scsi0 local-lvm:vm-9000-disk-0 --boot order=scsi0 \
  --ide2 local-lvm:cloudinit --serial0 socket --vga serial0
qm resize 9000 scsi0 60G
qm template 9000
```

**API user and token for automation**

```bash
pveum user add molecule@pve --comment "Ansible Molecule automation"
pveum acl modify /vms                    --users molecule@pve --roles PVEVMAdmin
pveum acl modify /storage/local-lvm      --users molecule@pve --roles PVEDatastoreUser
pveum acl modify /sdn/zones/localnetwork --users molecule@pve --roles PVESDNUser
pveum user token add molecule@pve ci --privsep 0     # copy the "value": it's shown only once
```

`--privsep 0` makes the token inherit the user's permissions. With privilege separation on, the
token has no permissions until you grant them to the token itself.

### 5.2 Create VMs and deploy

**Option A (testing): Molecule creates, deploys, verifies and destroys.** See [section 9](#9-test-with-molecule-proxmox).

**Option B (long-lived VMs): clone by hand, then run the quick start.**

```bash
# on the Proxmox host; key = the control node's public key
for spec in "9101 posit-workbench 192.168.1.51 6" "9102 posit-connect 192.168.1.52 4"; do
  set -- $spec
  qm clone 9000 $1 --name $2 --full
  qm set $1 --cores $4 --memory 8192 --ciuser ansible --sshkeys /root/ctl_id_ed25519.pub \
    --ipconfig0 ip=$3/24,gw=192.168.1.1 --nameserver 192.168.1.1
  qm start $1
done
```

Then follow [section 4](#4-deploy-to-any-existing-vms-quick-start) with `ansible_user=ansible`.

---

## 6. Azure

### 6.1 Create the VMs (Azure CLI)

```bash
RG=posit-poc; LOC=eastus; ADMIN=azureuser
az group create -n $RG -l $LOC
az network vnet create -g $RG -n posit-vnet --address-prefix 10.10.0.0/16 \
  --subnet-name posit-subnet --subnet-prefix 10.10.1.0/24
az network nsg create -g $RG -n posit-nsg
MYIP=$(curl -s https://ifconfig.me)
az network nsg rule create -g $RG --nsg-name posit-nsg -n ssh-posit --priority 100 \
  --source-address-prefixes $MYIP/32 --destination-port-ranges 22 8787 3939 --access Allow --protocol Tcp

for vm in posit-workbench posit-connect; do
  az vm create -g $RG -n $vm --image Ubuntu2404 --size Standard_D4s_v5 \
    --admin-username $ADMIN --ssh-key-values ~/.ssh/id_ed25519.pub \
    --vnet-name posit-vnet --subnet posit-subnet --nsg posit-nsg \
    --os-disk-size-gb 64 --public-ip-sku Standard
done
az vm list-ip-addresses -g $RG -o table
```

Note: This command assumes you already have an SSH key pair at `~/.ssh/id_ed25519.pub`. If you don't already have SSH keys configured, replace `--ssh-key-values ~/.ssh/id_ed25519.pub` with `--generate-ssh-keys`. Azure CLI will automatically generate an SSH key pair for you if one doesn't already exist.

### 6.2 Deploy

Put the IPs in `inventory.ini` with `ansible_user=azureuser`, then run [section 4](#4-deploy-to-any-existing-vms-quick-start).

IMPORTANT: When configuring Posit Connect as a publishing destination within Posit Workbench, you must use the private IP address of the Posit Connect VM.

Use the private IP (e.g. `10.10.1.x`), not the public IP.

The Workbench-to-Connect communication should take place over the Azure Virtual Network. The public IP is not the address to use for this internal connection.

**Notes**
- Restrict the NSG to your IP or VPN range. Don't expose 8787/3939 to the internet. For production,
  put an Application Gateway or reverse proxy with TLS in front.
- Use private IPs from a control node inside the VNet (e.g. a self-hosted Azure DevOps agent) instead of public IPs.
- Dynamic inventory: `azure.azcollection.azure_rm` with tags to group hosts into `workbench` and `connect`.
- Clean up: `az group delete -n $RG --yes`.

---

## 7. AWS

### 7.1 Create the VMs (AWS CLI)

```bash
REGION=us-east-1; KEY=posit-poc
AMI=$(aws ssm get-parameter --region $REGION \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query Parameter.Value --output text)
aws ec2 import-key-pair --region $REGION --key-name $KEY \
  --public-key-material fileb://~/.ssh/id_ed25519.pub
SG=$(aws ec2 create-security-group --region $REGION --group-name posit-poc \
  --description "Posit POC" --query GroupId --output text)
MYIP=$(curl -s https://checkip.amazonaws.com)
for p in 22 8787 3939; do
  aws ec2 authorize-security-group-ingress --region $REGION --group-id $SG \
    --protocol tcp --port $p --cidr $MYIP/32
done

for name in posit-workbench posit-connect; do
  aws ec2 run-instances --region $REGION --image-id $AMI --instance-type m6i.xlarge \
    --key-name $KEY --security-group-ids $SG \
    --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=64,VolumeType=gp3}' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$name},{Key=posit_role,Value=${name#posit-}}]"
done
aws ec2 describe-instances --region $REGION --filters Name=tag:Name,Values=posit-* \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`]|[0].Value,PublicIpAddress]' --output table
```

### 7.2 Deploy

Put the IPs in `inventory.ini` with `ansible_user=ubuntu`, then run [section 4](#4-deploy-to-any-existing-vms-quick-start).

**Notes**
- Without `--subnet-id`, instances land in the default VPC. Use private subnets plus a bastion or SSM for production.
- Dynamic inventory: `amazon.aws.aws_ec2` with `keyed_groups` on the `posit_role` tag gives you `workbench` and `connect` groups.
- Put an ALB with an ACM certificate in front for TLS.
- Clean up: terminate the instances, then delete the security group and key pair.

---

## 8. GCP

### 8.1 Create the VMs (gcloud CLI)

```bash
PROJECT=<your-project>; ZONE=us-central1-a
gcloud config set project $PROJECT
MYIP=$(curl -s https://ifconfig.me)
gcloud compute firewall-rules create posit-poc --network default \
  --allow tcp:22,tcp:8787,tcp:3939 --source-ranges $MYIP/32 --target-tags posit

for vm in posit-workbench posit-connect; do
  gcloud compute instances create $vm --zone $ZONE --machine-type e2-standard-4 \
    --image-family ubuntu-2404-lts-amd64 --image-project ubuntu-os-cloud \
    --boot-disk-size 64GB --tags posit \
    --labels posit_role=${vm#posit-} \
    --metadata ssh-keys="ansible:$(cat ~/.ssh/id_ed25519.pub)"
done
gcloud compute instances list --filter="name~posit-"
```

### 8.2 Deploy

Put the external IPs in `inventory.ini` with `ansible_user=ansible`, then run [section 4](#4-deploy-to-any-existing-vms-quick-start).

**Notes**
- If your organization enforces OS Login, use `gcloud compute ssh` or OS Login keys instead of the `ssh-keys` metadata.
- Dynamic inventory: `google.cloud.gcp_compute` with `keyed_groups` on the `posit_role` label.
- Put an HTTPS load balancer or IAP in front for TLS and access control.
- Clean up: `gcloud compute instances delete posit-workbench posit-connect --zone $ZONE` and `gcloud compute firewall-rules delete posit-poc`.

---

## 9. Test with Molecule (Proxmox)

Molecule linked-clones two VMs from the template, runs `site.yml`, checks the result and deletes the VMs.

```bash
source ~/posit-poc.env
cd <repo>
LIC="workbench_license_key=$PWB_LICENSE connect_license_key=$PCT_LICENSE workbench_test_password=<password>"

molecule create                       # clone and boot the VMs
molecule converge -- -e "$LIC"        # deploy (re-run while iterating)
molecule verify                       # health checks
molecule login -h posit-workbench     # shell into a VM
molecule destroy                      # delete the VMs

molecule test -- -e "$LIC"                      # full cycle
molecule test --destroy=never -- -e "$LIC"      # keep the VMs for a demo
```

Before the first run, set these to match your environment:

| File | Setting |
|---|---|
| `~/posit-poc.env` | `PVE_HOST`, `PVE_NODE`, `PVE_USER`, `PVE_TOKEN_ID`, `PVE_TOKEN_SECRET` |
| `molecule/default/create.yml` | `pve_template`, `pve_gateway`, `pve_dns` |
| `molecule/default/molecule.yml` | each VM's `vmid`, `ip`, `cores`, `memory`, `groups` |

The control node's `~/.ssh/id_ed25519(.pub)` is injected into the VMs and used by Ansible.

## 10. After deployment

| | URL | First sign-in |
|---|---|---|
| Workbench | `http://<workbench-ip>:8787` | The test user (`posituser`) and the password you passed |
| Connect | `http://<connect-ip>:3939` | Create an account; the first account becomes the administrator |

Check licenses:

```bash
sudo rstudio-server license-manager status                  # Workbench
sudo /opt/rstudio-connect/bin/license-manager status        # Connect
```

Before production, plan for:
- TLS/SSL: HTTPS must be enabled with a trusted SSL certificate for Positron Pro and VS Code sessions to work correctly. Some IDE functionality, including web views, will not work as expected without HTTPS and a trusted certificate.
- SSO (SAML/OIDC)
- Posit Package Manager
- backups of `/var/lib/rstudio-connect` and user home directories
- monitoring

See the [Posit Team documentation](https://docs.posit.co/posit-team/).

### 10.1 Licence deactivation (before destroying VMs)

IMPORTANT: Always deactivate your Posit licences before destroying
the Workbench or Connect VMs. Failing to do so may leave your
licence activation allocated to a server that no longer exists.

For licence keys, run the following commands on the respective servers:

Workbench:
    `sudo rstudio-server license-manager deactivate`

Connect:
    `sudo /opt/rstudio-connect/bin/license-manager deactivate`

## 11. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Non integer values in LooseVersion (' 3.10.4')` | Space after `>=` in `requirements.yml` | Write `">=3.10.4"` |
| `No such file or directory: 'pvesh'` | `PVE_HOST` empty | `source ~/posit-poc.env` |
| `'!ci' is not a valid token ID` | `PVE_USER` empty | Add `export PVE_USER=molecule@pve` |
| `401 invalid token value` | Wrong or empty `PVE_TOKEN_SECRET` | Recreate the token, update the env file |
| `VM with name ... does not exist` (template exists) | Token can't see VM 9000 | Token created with privilege separation, or missing ACL/pool membership |
| `403 Forbidden` / `SDN.Use` | Missing permission | Re-run the `pveum acl modify` commands |
| `Permission denied (publickey)` | Wrong key injected | `--sshkeys` must contain the control node's public key; `-i` takes the private key |
| `cloud-init status --wait` returns rc 2 | Finished with warnings | The task accepts rc 0 and 2 |
| `!` in a `curl` command breaks the token | Bash history expansion | Run `set +H` |
| `playbook: site.yml could not be found` | `converge:` path is wrong | Create `site.yml` or fix the path |
| `Could not get lock /var/lib/dpkg/lock` | cloud-init still running | Wait, or rely on the cloud-init wait task |
