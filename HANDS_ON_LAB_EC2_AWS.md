# Hands-On Lab — Running Everything from an EC2 Instance

A step-by-step guide to doing this entire lab using a **cloud EC2 instance** as your
orchestration machine ("laptop"), instead of a local Windows/WSL setup. See
[HANDS_ON_LOG.md](HANDS_ON_LOG.md) for how the local-Windows run actually went (including
the problems hit) — this document is the plan for the EC2-based alternative.

## Why this is different from running it locally

Running from a real Linux EC2 instance sidesteps every Windows/WSL-specific issue hit in
the local run:
- No `ansible.cfg` "world writable directory" warning (that only happens because WSL
  mounts the Windows C: drive with loose permissions — a real Linux root volume doesn't
  have this problem).
- No WSL root-vs-normal-user shell confusion, no `/mnt/c/...` path translation.
- Terraform, AWS CLI, and Ansible all install and behave exactly as their own
  documentation describes, since it's native Linux.

The trade-off: this "orchestration" EC2 instance is a **5th billable instance**, separate
from the 4 the lab itself provisions, and you have to think about how it authenticates to
AWS (see Step 2).

---

## Step 0 — Launch the orchestration EC2 instance

From your AWS Console (or CLI), launch one small instance to act as your "cloud laptop":

- AMI: Ubuntu 24.04 LTS (matches the rest of the lab)
- Instance type: `t3.micro` (free-tier eligible)
- A security group allowing **inbound SSH (port 22) from your own IP** (so you can reach
  it) — no need to open it to `0.0.0.0/0` for this one, unlike the lab's own instances.
- Default outbound rules are fine (it needs to reach the AWS API, GitHub, and later the
  lab's control node over the internet).
- Attach a keypair you already control, so you can SSH into this box from your laptop.

> If you already have an EC2 Ubuntu instance running for other purposes, you can reuse it
> instead of launching a new one — nothing here requires a dedicated box, just a working
> Ubuntu shell with internet access.

---

## Step 1 — Install prerequisites (all native, no WSL needed)

SSH into the orchestration instance, then:

```bash
sudo apt update
sudo apt install -y software-properties-common git unzip curl

# Ansible
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
ansible --version

# Terraform
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform -version

# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws --version
```

---

## Step 2 — Give this instance AWS credentials

Two options — pick one:

**Option A — IAM instance role (recommended).** Attach an IAM role to this EC2 instance
(via the AWS Console → instance → Actions → Security → Modify IAM role) with permissions
to create EC2 instances, security groups, and key pairs (e.g. `AmazonEC2FullAccess` for lab
purposes). No keys to manage at all — the instance authenticates automatically via its
instance metadata. Confirm it's attached:
```bash
aws sts get-caller-identity
```
This should immediately return an identity with **no** `aws configure` step needed.

**Option B — Access keys.** If you'd rather not attach a role:
```bash
aws configure
# Access Key ID, Secret Access Key, region: us-west-2
aws sts get-caller-identity
```

> From the local run: if this ever returns `InvalidClientTokenId` even after configuring
> correctly, check `aws configure list` — it shows whether credentials are actually coming
> from the config file or from a stray environment variable overriding it.

---

## Step 3 — Clone the repo

```bash
git clone https://github.com/sumeetgaurav/ansible-lab.git
cd ansible-lab
```

(This fork already has the `terraform/terra-key-ansible.pub` tracking bug fixed — the file
is now gitignored, so a fresh clone here won't repeat the "wrong public key baked into
every instance" problem hit during the local run.)

---

## Step 4 — Generate your own SSH keypair, on this instance

```bash
mkdir -p ~/keys
ssh-keygen -t rsa -b 4096 -f ~/keys/terra-key-ansible -N ""
chmod 600 ~/keys/terra-key-ansible
cp ~/keys/terra-key-ansible.pub terraform/terra-key-ansible.pub
```

Since the repo is on this instance's own native filesystem (not a Windows mount), there's
no need to `cp` across a mount boundary like in the WSL run — it's all one filesystem.

---

## Step 5 — Provision the infrastructure

```bash
cd terraform
terraform init
terraform plan        # review: should show 4 instances + 1 SG + 1 key pair + 2 local files to add
```

Confirm the plan looks right, then:
```bash
terraform apply -auto-approve
terraform output
cd ..
```

---

## Step 6 — Bootstrap the control node

No WSL hop needed this time — you're already in the right Linux shell:

```bash
ansible-playbook -i inventories/dev/bootstrap.ini playbooks/bootstrap_control_node.yml --syntax-check
ansible -i inventories/dev/bootstrap.ini control -m ping
ansible-playbook -i inventories/dev/bootstrap.ini playbooks/bootstrap_control_node.yml
```

If this is a brand-new SSH host, you may still be prompted to accept its host key the
first time — type `yes`, or set `export ANSIBLE_HOST_KEY_CHECKING=False` beforehand to skip
the prompt non-interactively.

---

## Step 7 — SSH into the control node and verify

```bash
ssh -i ~/keys/terra-key-ansible ubuntu@<control_node_public_ip>
cd ansible-in-one-shot
ansible all -m ping
```

Expect 4x `pong` (control + 3 workers). Unlike the WSL run, `ansible.cfg` will be picked up
with **no warning** here too, both on this orchestration box and on the control node,
since neither is on a Windows-mounted filesystem.

---

## Step 8 — Run the course modules

Same as the local run, all from inside the control node SSH session:

```bash
ansible-playbook modules/01-basics/01_ping.yml
# ... through modules/06-vault, per STUDY_CHECKLIST.md
```

---

## Step 9 — Cleanup (two things to destroy, not one)

```bash
# 1. The 4 lab instances (run from the orchestration box, in terraform/)
terraform destroy -auto-approve
```

Then, separately — **don't forget this one** — stop or terminate the orchestration EC2
instance itself from Step 0 if you're done for the day, since it's a 5th instance billing
independently of the lab's own 4.

---

## Quick comparison with the local (WSL) run

| | Local (Windows + WSL) | EC2 orchestration box |
|---|---|---|
| Ansible install location | Inside WSL only | Native on the instance |
| `ansible.cfg` auto-discovery | Broken on `/mnt/c/...` (world-writable warning) | Works normally |
| AWS credentials | `aws configure` on native Windows, env-var conflicts possible | IAM role (no keys) or `aws configure` |
| Extra billable resources | None (your own laptop) | +1 EC2 instance for orchestration |
| Path gotchas | WSL↔Windows path translation, root-vs-user shell mixups | None — single Linux filesystem throughout |
