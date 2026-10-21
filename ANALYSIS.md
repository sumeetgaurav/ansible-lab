# Repo Analysis

**One-liner:** An end-to-end, hands-on Ansible training course that uses Terraform to spin up a real 4-node AWS lab (1 control node + 3 OS-diverse workers) and teaches Ansible fundamentals through 6 progressive modules plus 3 production-style roles.

## What this repo achieves

- **Provisions real infrastructure**: `terraform/` creates 4 free-tier EC2 instances in AWS (`control-node-ubuntu`, `worker-ubuntu`, `worker-redhat`, `worker-amazon`) and auto-generates Ansible inventory files from the Terraform state.
- **Bootstraps a dedicated control node**: `playbooks/bootstrap_control_node.yml` is run once from your laptop to install Ansible on the control node, push SSH keys, clone the repo there, and hand off the teaching inventory — mirroring real-world setups where a control node (not a laptop) manages fleets.
- **Teaches Ansible step-by-step** via `modules/01-basics` → `06-vault`: ad-hoc commands, variables/facts, Jinja2 templates & handlers, loops/conditionals/blocks/tags, roles & `site.yml`, and Ansible Vault for secrets — each with its own README and runnable `.yml` examples.
- **Demonstrates production-grade reusable roles** in `roles/`: `common` (baseline OS config across Debian/RedHat families), `docker` (OS-specific install logic for Ubuntu/RHEL/Amazon Linux, including a kernel-module/reboot fix for RHEL), and `nginx` (templated web server deploy).
- **Supports multi-OS automation**: inventory groups (`ubuntu`, `redhat`, `amazon`, `workers`, `control`) and `group_vars`/`host_vars` let the same playbooks adapt behavior per OS family.
- **Includes a self-paced learning path**: `STUDY_CHECKLIST.md` (7-day plan with "break it on purpose" exercises) and `docs/` (ad-hoc command cheat sheet, manual setup walkthrough) for learners going solo instead of in the live 4-hour session.
- **Enforces quality**: `.ansible-lint` config (FQCN module names, truthy enforcement) and `ansible.cfg` set sane defaults for the whole course.

## Quick structure map

| Path | Purpose |
|---|---|
| `terraform/` | AWS infra (EC2, security group) + auto-generates `inventories/dev/*.ini` |
| `playbooks/bootstrap_control_node.yml` | One-time laptop → control-node handoff |
| `inventories/dev/` | Generated inventories + group/host vars |
| `modules/01-basics` … `06-vault` | The 6-part course curriculum (run these in order) |
| `roles/common, docker, nginx` | Reusable, production-style roles tied together by `modules/05-roles/site.yml` |
| `docs/`, `STUDY_CHECKLIST.md` | Reference material and self-study plan |
