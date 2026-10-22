# Hands-On Lab Log — Ansible In One Shot

A step-by-step record of actually running this lab end-to-end on real AWS infrastructure
from a Windows 11 laptop, including the real problems hit along the way and how they were
fixed. Written as a reference for redoing this (or explaining it to someone else) — not a
replacement for [STUDY_CHECKLIST.md](STUDY_CHECKLIST.md) or [README.md](README.md), which
describe the lab as designed.

## Environment

- Laptop: Windows 11. Terraform and AWS CLI installed **natively** on Windows.
- Ansible does **not** run on native Windows, so it was installed inside **WSL (Ubuntu)**
  and all `ansible` / `ansible-playbook` commands were run from a WSL bash shell.
- The repo lives on the Windows filesystem (`C:\dev-ops\ai-powered\ansible`), accessible
  from WSL at `/mnt/c/dev-ops/ai-powered/ansible`.

---

## Step 1 — Install prerequisites

On Windows (already present): Terraform, AWS CLI v2.

In WSL Ubuntu:
```bash
sudo apt update
sudo apt install -y software-properties-common git
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
ansible --version
```

### Gotcha: stale `AWS_*` environment variables shadowing real credentials

Running `aws sts get-caller-identity` kept failing with `InvalidClientTokenId`, **even
after** running `aws configure` with valid keys. Root cause: old `AWS_Access_Key_ID`,
`AWS_Secret_Access_Key`, and `AWS_DEFAULT_REGION` environment variables were already set
(confirmed via `aws configure list`, which showed the credential source as `env`, not
`config-file`). The AWS CLI/SDK credential chain checks environment variables **before**
the `~/.aws/credentials` file, so the stale/invalid env vars always won regardless of what
`aws configure` saved.

Fix:
1. Cleared the vars for the current session and permanently at the User level:
   ```powershell
   Remove-Item Env:AWS_Access_Key_ID, Env:AWS_Secret_Access_Key, Env:AWS_DEFAULT_REGION -ErrorAction SilentlyContinue
   [Environment]::SetEnvironmentVariable('AWS_Access_Key_ID', $null, 'User')
   [Environment]::SetEnvironmentVariable('AWS_Secret_Access_Key', $null, 'User')
   [Environment]::SetEnvironmentVariable('AWS_DEFAULT_REGION', $null, 'User')
   ```
2. Restarted the terminal/Claude Code session so the process picked up the cleared
   environment (clearing a User env var does not affect already-running processes).
3. Verified: `aws sts get-caller-identity` returned the real account identity.

**Lesson:** if `aws configure` seems to have no effect, check `aws configure list` — it
tells you exactly which source (`env`, `config-file`, `env`, `None`) each credential/region
value is coming from.

---

## Step 2 — Generate the SSH keypair

In WSL:
```bash
mkdir -p ~/keys
ssh-keygen -t rsa -b 4096 -f ~/keys/terra-key-ansible -N ""
chmod 600 ~/keys/terra-key-ansible
cp ~/keys/terra-key-ansible.pub /mnt/c/dev-ops/ai-powered/ansible/terraform/terra-key-ansible.pub
```

### Gotcha: the repo already had someone else's public key committed

`terraform/terra-key-ansible.pub` turned out to already exist in git history (added in an
earlier commit, `75ed368 "added prod code"`), containing the **course author's** public
key (`shubhamlondhe@...`). `.gitignore` only excludes the private key filenames, not this
`.pub` file — so it was never meant to survive in the repo, but it did.

The first `terraform apply` ran successfully using that stale `.pub` file, which meant all
4 EC2 instances booted with the wrong public key baked into `~/.ssh/authorized_keys` —
nobody but the original author could SSH in.

Fix:
1. Overwrote `terraform/terra-key-ansible.pub` with the real, freshly generated key (copied
   the file directly rather than retyping it — a manually retyped RSA public key is long
   enough that a single mistyped character silently breaks it).
2. Ran `terraform plan` — it showed only the `aws_key_pair` resource being replaced, **not**
   the EC2 instances (because `aws_instance.key_name` references the key pair's `key_name`
   attribute, which doesn't change — only its `public_key`/`fingerprint` do). That matters:
   replacing just the key pair object does **not** retroactively change the
   `authorized_keys` file already written to a running instance at boot time.
3. Did a full `terraform destroy -auto-approve` followed by `terraform apply -auto-approve`
   to force all 4 instances to boot fresh with the correct key.

**Lesson:** an EC2 instance's SSH key is injected once, at first boot. Changing the
`aws_key_pair` resource afterward does nothing for already-running instances — they must be
replaced.

---

## Step 3 — Provision infrastructure

```bash
cd terraform
terraform init
terraform plan      # reviewed before applying — real, billable resources
terraform apply -auto-approve
terraform output
```

Result: 4 EC2 instances in `us-west-2` (t3.micro):

| Host | OS | SSH user |
|---|---|---|
| control-node-ubuntu | Ubuntu 24.04 | ubuntu |
| worker-ubuntu | Ubuntu 24.04 | ubuntu |
| worker-redhat | RHEL 9 | ec2-user |
| worker-amazon | Amazon Linux 2023 | ec2-user |

Terraform auto-generated `inventories/dev/bootstrap.ini` (public IP, for the one-time
bootstrap) and `inventories/dev/hosts.ini` (private IPs, for use on the control node).

---

## Step 4 — Bootstrap the control node

Run **from WSL** (needs Ansible, which only exists there):
```bash
cd /mnt/c/dev-ops/ai-powered/ansible
ansible-playbook -i inventories/dev/bootstrap.ini playbooks/bootstrap_control_node.yml --syntax-check
ansible -i inventories/dev/bootstrap.ini control -m ping
ansible-playbook -i inventories/dev/bootstrap.ini playbooks/bootstrap_control_node.yml
```

### Gotcha: `ansible.cfg` silently ignored on the Windows-mounted drive

Running Ansible from `/mnt/c/...` inside WSL triggers:
```
[WARNING]: Ansible is being run in a world writable directory (/mnt/c/...), ignoring it as an ansible.cfg source.
```
WSL mounts the Windows C: drive with permissions that look world-writable to Linux, and
Ansible refuses to trust a config file sitting in such a directory (security hardening).
This wasn't a blocker for bootstrap specifically because `bootstrap.ini` sets the SSH key
and Python interpreter explicitly as inventory vars rather than relying on `ansible.cfg`.
It only matters if you try to run the course modules directly from WSL against `hosts.ini`
instead of from the control node itself — once you SSH onto the actual control node
(native Linux filesystem, not a Windows mount), `ansible.cfg` loads normally and every
module command works exactly as the README describes (no explicit `-i` needed).

Also set explicitly to avoid a hanging "yes/no" host-key prompt on the first connection to
a brand-new instance:
```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```

### Gotcha: WSL root shell vs. normal user shell

While fixing a forgotten WSL password with `wsl -d Ubuntu -u root`, a later `ssh -i
~/keys/terra-key-ansible ...` failed with `Permission denied (publickey)` because the
shell was still running as `root` (home `/root`), not the normal user `dell` (home
`/home/dell`) where the key actually lives. `~` resolves per-user — always check `whoami`
before trusting a `~/...` path.

Bootstrap playbook result: installed Ansible on the control node, pushed the SSH private
key there, cloned the course repo, installed the `requirements.yml` Galaxy collections, and
copied the teaching inventory (`hosts.ini`) into place — 10 tasks, 7 changed, 0 failed.

---

## Step 5 — Verify connectivity

SSH into the control node and ping everything:
```bash
ssh -i ~/keys/terra-key-ansible ubuntu@<control_node_public_ip>
cd ansible-in-one-shot
ansible all -m ping
```
Result: `pong` from `control-node-ubuntu`, `worker-ubuntu`, `worker-redhat`, and
`worker-amazon` — all 4 nodes reachable.

---

## Step 6 — Module 1: Basics

```bash
ansible-playbook modules/01-basics/01_ping.yml
ansible-playbook modules/01-basics/02_gather_facts.yml
ansible-playbook modules/01-basics/03_install_packages.yml
ansible-playbook modules/01-basics/04_file_operations.yml
ansible-playbook modules/01-basics/05_services.yml
```

### Gotcha: fresh Ubuntu AMI has an empty apt package index

`03_install_packages.yml` (installs `tree`, `wget`, `vim` via the generic `package`
module) failed only on `worker-ubuntu`:
```
fatal: [worker-ubuntu]: FAILED! => {"changed": false, "msg": "No package matching 'tree' is available"}
```
`control-node-ubuntu` didn't hit this because the bootstrap playbook already ran `apt
update` there. `worker-ubuntu` never had its apt cache refreshed — cloud AMIs commonly ship
with `/var/lib/apt/lists` cleared out to shrink the image.

Fix:
```bash
ansible workers -b -a "apt-get update" --limit worker-ubuntu
ansible-playbook modules/01-basics/03_install_packages.yml
```

Then re-ran the same playbook a second time to prove idempotency:
```bash
ansible-playbook modules/01-basics/03_install_packages.yml
```
First run after the fix: `changed=1..3` per host. Second run: `changed=0` on **all** hosts —
confirms the playbook correctly detects "already installed" and does nothing further.

---

## Step 7 — Module 2: Variables & Facts

```bash
ansible-playbook modules/02-variables-and-facts/01_simple_variables.yml
ansible-playbook modules/02-variables-and-facts/02_register_debug.yml
ansible-playbook modules/02-variables-and-facts/03_host_group_vars.yml
ansible-playbook modules/02-variables-and-facts/04_facts_conditionals.yml
```

`01_simple_variables.yml` and `03_host_group_vars.yml` only print values via `debug` —
nothing on disk changes, so there's nothing to independently verify on the worker beyond
matching the printed output against `inventories/dev/group_vars/*.yml`.
`04_facts_conditionals.yml` is the one that changes real state (runs `apt update` on
Debian-family hosts) and reports real facts (`os_family`, `memtotal_mb`) worth
cross-checking.

### Validating facts directly on a worker (not just trusting Ansible's `debug` output)

Two ways:

**A. Ad-hoc commands from the control node** (runs the real OS command, bypassing the
facts/debug layer):
```bash
ansible workers -a "cat /etc/os-release"
ansible workers -a "free -m"
ansible redhat -a "which dnf"
ansible ubuntu -a "which apt"
```

**B. Direct SSH into a worker**, bypassing Ansible entirely. The control node already has
the private key (bootstrap pushed it there) and reaches workers over their **private**
IPs:
```bash
cat inventories/dev/hosts.ini     # confirm private IPs + SSH users
ssh -i ~/keys/terra-key-ansible ec2-user@<worker-redhat-private-ip>
cat /etc/os-release
free -m
which dnf
rpm -q tree wget vim               # confirms Module 1's installs actually landed
exit
```
(Swap `ec2-user`/`rpm -q` for `ubuntu`/`dpkg -l` on `worker-ubuntu`.)

---

## Key takeaways so far

1. **Credential source matters, not just whether you ran `aws configure`** — environment
   variables silently beat the credentials file. `aws configure list` shows the real source.
2. **A `.pub` file committed to a shared repo is a real security/operability bug** — anyone
   who runs `terraform apply` as-is provisions instances they personally cannot access.
   Always generate your own keypair and verify the `.pub` file in the repo is actually
   yours before `apply`.
3. **EC2 SSH keys are baked in at boot, not live-synced** — replacing the Terraform key pair
   resource doesn't fix already-running instances; they must be replaced.
4. **WSL's `/mnt/c/...` mount breaks Ansible's `ansible.cfg` auto-discovery** (world-writable
   directory check) — not an issue once you're working from the control node's own native
   filesystem.
5. **Idempotency is verifiable, not just a claim** — running the same playbook twice and
   diffing the `changed=` counts is the actual proof.
