# AWS EC2 Provisioning and Configuration Automation Using Ansible

## Project Overview

This project is a hands-on implementation of AWS EC2 provisioning and Linux server automation using Ansible. The objective was to understand how to create cloud instances, configure SSH key-based authentication, and automate administrative tasks across multiple Linux servers without performing every operation manually.

I organized the project into separate Ansible playbooks so that each task can be executed and tested independently. I also explored Ansible roles for Nginx installation and service management, along with troubleshooting common issues related to SSH connectivity, inventory configuration, and package installation.

The project is built around three main tasks:

1. Provision three EC2 instances using an Ansible loop: two Ubuntu instances and one CentOS-compatible instance.
2. Configure SSH key-based authentication between the Ansible control node and the managed instances.
3. Use Ansible facts and conditional execution to shut down Ubuntu instances while skipping non-Ubuntu systems.

An additional playbook demonstrates Nginx installation and service management on Ubuntu servers.

## Project Objectives

- Automate the creation of multiple AWS EC2 instances using Ansible.
- Understand the role of the Ansible control node and managed nodes.
- Configure SSH key-based authentication for remote server administration.
- Create and manage Ansible inventory files and YAML playbooks.
- Use loops to perform repetitive provisioning tasks.
- Use gathered system facts and `when` conditions to execute tasks based on the operating system.
- Install and manage Nginx using an Ansible role.
- Troubleshoot common infrastructure automation issues.
- Maintain project files and documentation in GitHub.

## Technologies and Tools

| Technology | Purpose |
|---|---|
| AWS EC2 | Provisioning and managing virtual servers |
| Ubuntu Linux | Ansible control environment and managed servers |
| CentOS-compatible Linux | Practicing automation across different Linux distributions |
| Ansible | Configuration management and task automation |
| YAML | Writing Ansible playbooks and configuration files |
| SSH | Secure remote access to EC2 instances |
| Ansible Inventory | Defining target hosts and connection settings |
| Ansible Facts | Identifying operating-system information |
| Ansible Roles | Organizing reusable Nginx configuration tasks |
| Nginx | Web server installation and service management |
| Git and GitHub | Version control and project documentation |

## Architecture

The project uses an Ansible control node to communicate with AWS and manage remote EC2 instances.

```text
                 Developer / Local Ubuntu
                           |
                           v
                  Ansible Control Node
                           |
              +------------+------------+
              |                         |
              v                         v
       AWS API / EC2               SSH Connection
       Provisioning                Key-Based Access
              |                         |
              v             +-----------+-----------+
        +-----------+        |           |           |
        | AWS EC2   |        v           v           v
        | Instances |    Ubuntu EC2  Ubuntu EC2  CentOS EC2
        +-----------+        |           |           |
                             +-----------+-----------+
                                         |
                                         v
                              Ansible Inventory and
                                  Playbooks
                                         |
                           +-------------+-------------+
                           |             |             |
                           v             v             v
                      SSH Setup     OS Condition   Nginx Role
                                    and Shutdown   (Ubuntu)
```

**Note:** The AWS API is used for EC2 provisioning, while SSH is used for configuration tasks on managed instances. The local control machine must have AWS API access and a valid network route to the target servers.

## Project Structure

```text
ansible-ec2-provisioning-automation/
│
├── README.md
├── .gitignore
├── ansible.cfg
├── requirements.yml
│
├── vars/
│   └── aws.yml.example
│
├── inventory/
│   └── hosts.example.ini
│
├── playbooks/
│   ├── 01_create_ec2.yml
│   ├── 02_configure_ssh_keys.yml
│   ├── 03_shutdown_ubuntu.yml
│   └── 04_install_nginx.yml
│
├── roles/
│   └── nginx/
│       ├── defaults/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       └── files/
│           └── index.html
│
└── docs/
    └── troubleshooting.md
```

The example configuration files provide a starting point. Environment-specific inventory and AWS configuration files are created locally and excluded from version control.

## Prerequisites

Before running the playbooks, ensure the following requirements are met:

- An AWS account with permission to manage the required EC2 resources.
- An Ubuntu system with Ansible installed.
- AWS API credentials or an appropriate IAM role for EC2 provisioning.
- A valid EC2 key pair for initial SSH access.
- Current AMI IDs for Ubuntu and the chosen CentOS-compatible distribution in the selected AWS region.
- A VPC, subnet, and security group configured for the intended connectivity.
- Python dependencies and Ansible collections required by the playbooks.
- A GitHub repository for storing the project source code.

The AMI IDs, key-pair name, subnet ID, security-group ID, and region must be configured for your own AWS environment before provisioning.

## Task 1: Provision Three EC2 Instances Using Ansible Loops

**Playbook:** `playbooks/01_create_ec2.yml`

The first task automates the creation of three EC2 instances using the `amazon.aws.ec2_instance` module and an Ansible loop.

The intended configuration is:

- Two Ubuntu EC2 instances.
- One CentOS-compatible EC2 instance.
- A common instance type and AWS region.
- A specified EC2 key pair and security group.
- Tags to identify the instances as part of the Ansible project.

The playbook uses `connection: local` because instance creation is performed through the AWS API from the Ansible control node. It does not require an SSH connection to the instances before they exist.

### Configuration

The file `vars/aws.yml.example` contains the variables required for provisioning.

Create your local configuration file:

```bash
cp vars/aws.yml.example vars/aws.yml
```

Update the AMI IDs, AWS region, key-pair name, subnet, and security-group ID before running the playbook.

### Execution

Check the playbook syntax:

```bash
ansible-playbook playbooks/01_create_ec2.yml --syntax-check
```

Run the provisioning playbook:

```bash
ansible-playbook playbooks/01_create_ec2.yml
```

After execution, verify the instance states, instance IDs, tags, and networking configuration in the AWS Management Console.

### What I learned

This task helped me understand how Ansible loops reduce repetitive configuration, how AWS modules interact with EC2, and why cloud provisioning requires correct region-specific AMI IDs and AWS permissions.

## Task 2: Configure Passwordless SSH Authentication

**Playbook:** `playbooks/02_configure_ssh_keys.yml`

SSH key-based authentication allows the Ansible control node to connect to managed instances without entering the target account password for every connection.

During this task, I worked with SSH key generation, public-key installation, target usernames, and private-IP connectivity.

### Step 1: Generate an SSH key pair

On the Ansible control node, generate an Ed25519 key pair:

```bash
ssh-keygen -t ed25519
```

This creates a private key and a public key, normally located at:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

The private key must remain secure. Only the public key should be installed on the target instances.

### Step 2: Establish initial access

Before installing a new public key, the control node must already have an accepted way to connect to the target. For an EC2 instance, this may be the original EC2 key pair or another approved access method.

The initial SSH connection must use the correct username and private key for the target image.

### Step 3: Install the public key

The project uses the `ansible.posix.authorized_key` module to install the control node's public key into the managed user's `authorized_keys` file.

The public key is read on the control node, then added to the target instance by Ansible. The private key is not copied to the target.

### Step 4: Verify SSH access

Test the connection:

```bash
ssh ubuntu@TARGET_PRIVATE_IP
```

Replace `TARGET_PRIVATE_IP` with the actual private IP of the target instance.

After connecting, verify the remote session:

```bash
whoami
hostname
```

Exit the session:

```bash
exit
```

The expected result is a successful SSH login using the configured key without being prompted for the target account password.

### Step 5: Verify Ansible connectivity

Once the inventory is configured:

```bash
ansible-inventory -i inventory/hosts.ini --graph
```

Test the managed hosts:

```bash
ansible linux_nodes -i inventory/hosts.ini -m ping
```

A successful connection returns `SUCCESS` and `"ping": "pong"` for each reachable target.

### What I learned

This task helped me understand public and private SSH keys, the purpose of `authorized_keys`, the difference between initial access and subsequent key-based login, and how Ansible uses SSH to communicate with managed servers.

## Task 3: Shut Down Ubuntu Instances Using Ansible Conditionals

**Playbook:** `playbooks/03_shutdown_ubuntu.yml`

The third task demonstrates operating-system-specific automation.

Instead of running the shutdown command on every managed host, the playbook gathers system facts and uses a `when` condition to target Ubuntu systems only.

The condition is:

```yaml
when: ansible_distribution == "Ubuntu"
```

### How it works

1. Ansible connects to the managed hosts.
2. Facts are gathered to identify the operating system.
3. The shutdown task runs only when the detected distribution is Ubuntu.
4. For non-Ubuntu hosts, the shutdown task is skipped.

The playbook also includes a message task to make the decision visible during execution.

### Execution

Review the inventory and confirm that it contains the intended Ubuntu and CentOS-compatible targets.

Check syntax:

```bash
ansible-playbook -i inventory/hosts.ini \
  playbooks/03_shutdown_ubuntu.yml --syntax-check
```

Run the playbook only when you intend to shut down the Ubuntu instances:

```bash
ansible-playbook -i inventory/hosts.ini \
  playbooks/03_shutdown_ubuntu.yml
```

### Expected behaviour

| Operating system | Expected action |
|---|---|
| Ubuntu | Shutdown task executes |
| CentOS-compatible Linux | Shutdown task is skipped |

The SSH connection may close when an Ubuntu machine shuts down. This is expected behaviour, not necessarily a playbook failure.

**Safety note:** This playbook shuts down the Ubuntu operating systems. Confirm the target hosts before execution and save any work first. Stopping the operating system does not necessarily stop AWS billing for attached storage or other associated resources.

### What I learned

This task helped me understand Ansible fact gathering, the `ansible_distribution` variable, conditional task execution, and how the same playbook can behave differently depending on the target operating system.

## Additional Task: Install and Start Nginx Using an Ansible Role

**Playbook:** `playbooks/04_install_nginx.yml`

As an additional exercise, I used an Ansible role to organize Nginx installation and service management on Ubuntu web servers.

The role separates configuration defaults, tasks, handlers, and website files.

### Main activities

- Refresh the APT package cache.
- Install Nginx.
- Deploy a custom HTML page.
- Ensure the Nginx service is running.
- Enable the service to start automatically when the system boots.
- Reload Nginx when the deployed page changes.

### Execution

Check the playbook syntax:

```bash
ansible-playbook -i inventory/hosts.ini \
  playbooks/04_install_nginx.yml --syntax-check
```

Run the playbook:

```bash
ansible-playbook -i inventory/hosts.ini \
  playbooks/04_install_nginx.yml
```

Test the web server from the control node:

```bash
curl http://TARGET_PRIVATE_IP/
```

The target must be reachable over the network, and the security group must permit HTTP traffic from the testing source.

### What I learned

This exercise introduced me to Ansible roles, reusable tasks, handlers, service management, package installation, and deploying a basic website through configuration automation.

## Troubleshooting and Lessons Learned

During development, I encountered several issues that helped me understand the relationship between SSH, networking, Ansible inventory, and Linux package management.

### 1. SSH: `Permission denied (publickey)`

This error means the target did not accept the provided SSH authentication.

I learned to check the SSH username, the correct EC2 key pair, the matching public key, and the target user's `authorized_keys` file.

The `ssh-copy-id` command also requires an existing authentication method to establish the initial connection.

### 2. SSH: `Connection timed out`

This can occur when the target is not reachable or does not respond to SSH traffic.

I learned to check the EC2 instance state, security-group rules, network ACLs, subnet routing, and whether TCP port 22 is permitted from the control node.

### 3. SSH: `Network is unreachable`

A private IP address such as `172.31.x.x` is normally reachable only from an appropriately connected network.

When working from a local computer, I learned that I must use a valid network route to the target VPC or connect through an appropriate public endpoint or secure access method.

### 4. Ansible: `skipping: no hosts matched`

This occurs when the playbook's `hosts` pattern does not match any hosts in the selected inventory.

I learned to verify the inventory file, group names, and command-line inventory path.

Useful checks include:

```bash
ansible-inventory -i inventory/hosts.ini --graph
```

```bash
ansible-playbook -i inventory/hosts.ini \
  playbooks/04_install_nginx.yml --syntax-check
```

### 5. APT: `404 Not Found`

The Nginx installation initially encountered package download errors because the requested package version was no longer available at the configured repository URL.

I learned to refresh the package index, avoid unnecessary version pinning, and verify the repository configuration when package installation fails.

More troubleshooting details are available in [`docs/troubleshooting.md`](docs/troubleshooting.md).

## Security Considerations

Security is important when automating cloud infrastructure.

- Keep SSH private keys outside the Git repository.
- Never commit AWS access keys, secret keys, passwords, or other credentials.
- Use an IAM role or secure credential provider with the minimum AWS permissions required.
- Restrict inbound SSH traffic to a trusted source or the appropriate control-node security group.
- Use the correct SSH username and key for each target operating system.
- Review shutdown and provisioning tasks before running them.
- Verify AWS resources after execution to avoid leaving unnecessary instances running.
- Keep local inventory and environment-specific configuration files excluded from version control.

## Validation and Project Evidence

To demonstrate the implementation, add screenshots of your own test results to the repository.

Recommended evidence:

1. The three EC2 instances in the AWS Console.
2. The Ansible inventory graph showing the managed hosts.
3. A successful SSH key-based login or Ansible ping result.
4. The shutdown playbook showing the Ubuntu condition and the skipped non-Ubuntu task.
5. The Nginx playbook recap and the custom webpage, if that task has been completed successfully.

Only include screenshots that represent actual test results. Hide sensitive information before publishing them.

## Conclusion

This project gave me practical experience with AWS EC2, Linux administration, SSH key-based authentication, and Ansible automation. I used loops for repetitive provisioning, inventories to organize managed hosts, and facts with conditional execution to target specific operating systems.

I also practiced organizing Nginx configuration into an Ansible role and troubleshooting issues involving SSH access, network reachability, inventory matching, and package installation.

The project represents my hands-on learning in infrastructure automation and provides a foundation for exploring more advanced DevOps practices.

## Author

**Srinivasu Polamuri**

GitHub: [srinivasupolamuri](https://github.com/srinivasupolamuri)

Project repository: [ansible-ec2-provisioning-automation](https://github.com/srinivasupolamuri/ansible-ec2-provisioning-automation)
