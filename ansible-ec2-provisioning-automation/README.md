# AWS EC2 Provisioning and Configuration Automation with Ansible

## Project overview

This hands-on project uses Ansible to provision and manage AWS EC2 instances, configure SSH key-based access, and perform operating-system-specific tasks. It reflects a lab built around three core tasks:

1. Launch three EC2 instances using an Ansible loop: two Ubuntu instances and one CentOS-compatible instance.
2. Configure passwordless SSH authentication from the Ansible control node to the managed instances.
3. Shut down Ubuntu instances only by gathering facts and applying an Ansible conditional.

The project also includes an optional Nginx role based on the user's earlier practice with package installation, service management, inventory files, and playbooks.

> **Important:** AWS AMI IDs are region-specific and change over time. The project uses variables for AMI IDs rather than hardcoding example IDs. Fill them with current, verified AMI IDs for your selected AWS region before provisioning.

## Architecture

```text
                         AWS account / VPC
                                 |
                      Ansible Control Node
                       Ubuntu + Ansible
                                 |
                  AWS API (instance provisioning)
                                 |
               +-----------------+-----------------+
               |                 |                 |
          Ubuntu EC2-1      Ubuntu EC2-2       CentOS EC2
               |                 |                 |
               +------ SSH key-based access ------+
                                 |
                     Ansible inventory/playbooks
                                 |
                    OS facts + Ubuntu conditional
                                 |
                 Ubuntu instances: shutdown task
                 CentOS instance: task skipped
```

## Skills demonstrated

- AWS EC2 provisioning and instance lifecycle concepts
- Ansible control node and managed nodes
- Ansible inventory files and YAML playbooks
- Ansible loops, variables, facts, and `when` conditionals
- SSH public-key authentication
- Linux/Ubuntu administration and basic CentOS-family awareness
- Nginx package and service management (optional role)
- Git/GitHub project organization and documentation
- Troubleshooting SSH, networking, inventory, and APT package errors

## Repository structure

```text
ansible-ec2-provisioning-automation/
├── README.md
├── .gitignore
├── ansible.cfg
├── requirements.yml
├── vars/
│   └── aws.yml.example
├── inventory/
│   └── hosts.example.ini
├── playbooks/
│   ├── 01_create_ec2.yml
│   ├── 02_configure_ssh_keys.yml
│   ├── 03_shutdown_ubuntu.yml
│   └── 04_install_nginx.yml
├── roles/
│   └── nginx/
│       ├── defaults/main.yml
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── files/index.html
└── docs/
    └── troubleshooting.md
```

## Prerequisites

- An AWS account and a selected region.
- An Ansible control node with Ansible installed.
- AWS API access for the control node. Prefer an attached IAM instance role with least-privilege permissions; do not store access keys in this repository.
- An EC2 key pair for the initial/bootstrap SSH connection.
- Current, verified AMI IDs for two Ubuntu instances and one CentOS-compatible instance in the chosen region.
- Security groups that permit SSH from the control node or another trusted source.
- Ansible collections and Python dependencies listed below.

## 1. Install dependencies

On the Ansible control node:

```bash
sudo apt update
sudo apt install -y python3-boto3 git
ansible-galaxy collection install -r requirements.yml
ansible --version
```

If Ansible uses a different Python interpreter, install `boto3` into that same interpreter/environment.

## 2. Configure AWS variables

Copy the example variables file:

```bash
cp vars/aws.yml.example vars/aws.yml
nano vars/aws.yml
```

Update the region, verified AMI IDs, key-pair name, and security-group ID. The file `vars/aws.yml` is ignored by Git. Never put AWS access keys or secret keys in it.

You can discover current AMIs in the AWS Console or through AWS Systems Manager public parameters where available. Ubuntu and CentOS-compatible AMIs differ by region, CPU architecture, and release. Ensure the AMIs match your chosen instance type and tutorial requirements.

## 3. Provision three EC2 instances

Review `playbooks/01_create_ec2.yml`. It uses `connection: local` because the playbook calls the AWS API from the control node, not SSH into the new machines.

Run:

```bash
ansible-playbook playbooks/01_create_ec2.yml
```

The playbook requests two Ubuntu instances and one CentOS-compatible instance using a loop. Verify the instance IDs, state, subnet, security groups, and tags in the AWS Console.

**Cost/safety:** EC2 instances and associated EBS volumes, public IPv4 addresses, and other resources may incur charges. Use an appropriate lab instance type, monitor billing, and terminate resources when you no longer need them. Do not run provisioning repeatedly without checking for existing instances.

## 4. Configure SSH key-based authentication

Passwordless SSH means key-based login without entering an account password for each connection. It does not mean disabling authentication.

The first connection still requires a bootstrap method. The private key corresponding to the EC2 key pair must be available securely on the Ansible control node, or another approved initial-access method must be used. The public key you want to install is supplied through `ansible_public_key_file`.

Edit a local inventory with the real private IPs and correct usernames for the chosen images. Ubuntu commonly uses `ubuntu`; CentOS images may use `centos` or another image-specific default user.

Then run:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/02_configure_ssh_keys.yml
```

The `ansible.builtin.authorized_key` task installs the control node's public key into each target user's `authorized_keys`. Keep the bootstrap private key out of the repository.

## 5. Shut down Ubuntu instances only

The playbook `playbooks/03_shutdown_ubuntu.yml` gathers facts and uses a `when` conditional checking `ansible_distribution == "Ubuntu"`. The shutdown task should run only on Ubuntu targets; the CentOS target should be skipped.

**Warning:** Running this playbook will stop the Ubuntu operating systems and disconnect SSH. Execute it only when you intend to stop those lab instances and have saved your work. A safer alternative for AWS labs is stopping instances through the EC2 API and explicitly selecting instance IDs/tags.

Run:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/03_shutdown_ubuntu.yml
```

Confirm the recap shows the shutdown task executed for Ubuntu hosts and skipped for the CentOS host. An SSH connection may close before a final result can be returned, because the machine is shutting down.

## 6. Optional: Install and start Nginx

After the SSH inventory is configured, the optional Nginx playbook installs Nginx on Ubuntu hosts, deploys a simple page, and ensures the service is enabled and running:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/04_install_nginx.yml --syntax-check
ansible-playbook -i inventory/hosts.ini playbooks/04_install_nginx.yml
```

Test from the control node:

```bash
curl http://TARGET_PRIVATE_IP/
```

For browser access, allow TCP port 80 from the intended test source in the target's security group. Do not open unnecessary ports to the internet.

## Troubleshooting notes

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for notes related to issues encountered during practice:
- `Permission denied (publickey)`
- `Connection timed out` and `Network is unreachable`
- `provided hosts list is empty` / `skipping: no hosts matched`
- APT `404 Not Found` while installing Nginx

## Evidence to add to GitHub

Capture your own terminal or AWS Console screenshots after testing:
1. EC2 instances created with the intended OS tags.
2. `ansible-inventory --graph` showing managed nodes.
3. successful SSH key-based login or `ansible.builtin.ping`.
4. shutdown playbook showing Ubuntu task execution and CentOS skip.
5. Nginx playbook recap and page response (if you complete the optional Nginx task).

Store sanitized screenshots in a `screenshots/` directory if desired. Hide account IDs, sensitive public IPs, private keys, tokens, and credentials.

## Resume wording (only claim tasks you have successfully tested)

**AWS EC2 Provisioning and Configuration Automation Using Ansible**  
Used Ansible loops and AWS modules to provision EC2 instances for Ubuntu and CentOS-compatible Linux environments. Configured SSH key-based access to managed nodes and created inventory files and playbooks to automate remote administration. Used gathered system facts and `when` conditionals to target Ubuntu instances for shutdown operations. Also practiced Nginx installation and service management through Ansible.

Adjust this summary to include only the components you actually completed and validated.

## Security checklist

- [ ] No `.pem` or private SSH key committed.
- [ ] No AWS access key, secret key, or password committed.
- [ ] `vars/aws.yml` and `inventory/hosts.ini` are ignored by Git.
- [ ] Security group limits SSH to a trusted source.
- [ ] AMI IDs have been verified for the selected region.
- [ ] Shutdown playbook has been reviewed before execution.
- [ ] README claims match actual test results.
