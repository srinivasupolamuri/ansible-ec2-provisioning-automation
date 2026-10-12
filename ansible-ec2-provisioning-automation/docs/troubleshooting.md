# Troubleshooting notes from the lab

## `Permission denied (publickey)`

SSH reached the target, but the target did not accept the key/user combination. Check the image's default SSH username, the key pair associated with the target, file permissions, and whether the matching public key is in the target user's `~/.ssh/authorized_keys`.

`ssh-copy-id` cannot bootstrap itself if no accepted authentication method exists. First connect with the target's original EC2 key or an approved access method, then install the Ansible control node public key.

## `Connection timed out` / `Network is unreachable`

Private EC2 IPs (for example, `172.31.x.x`) are not generally reachable directly from a home network. Run Ansible from a control node with a route to the target VPC, or connect using a target public IP with strict security-group rules. Check instance state, VPC/subnet routing, network ACLs, security-group TCP/22 ingress, and:

```bash
ip route
ip route get TARGET_PRIVATE_IP
```

## `provided hosts list is empty` / `skipping: no hosts matched`

Check the path and content of `inventory/hosts.ini`, and ensure the playbook's `hosts:` group matches an inventory group.

```bash
ansible-inventory -i inventory/hosts.ini --graph
ansible-playbook -i inventory/hosts.ini playbooks/04_install_nginx.yml --syntax-check
```

## APT `404 Not Found` during Nginx install

APT indexes can be stale, or a playbook can pin a package version that is no longer available. This role refreshes the APT cache and installs the repository's current `nginx` package rather than pinning a specific version. If errors persist, inspect `sudo apt update` output and repository configuration.

## Shutdown task disconnects from the host

That is expected when the operating system shuts down. The playbook uses asynchronous execution so Ansible can dispatch the shutdown without waiting for the SSH connection to remain alive. Run it only when you intend to shut down the Ubuntu lab instances.
