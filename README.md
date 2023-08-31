# rocky-bastion

Helper scripts and Ansible for quick setup of a bastion server.

* Designed to work with **Rocky Linux 9**
* Playbook self-configures the system it is run on

## Pre-flight setup

After provisioning a new system, the following one-time steps are needed.

### Update all packages and reboot

```bash
sudo dnf -y update
```

Reboot in order to use latest kernel and updated nftables.

```bash
sudo systemctl reboot
```

### Install git and Python requirements

```bash
sudo dnf -y install git python3-pip
```

### Prepare venv and install Ansible

Clone this repo, then cwd to it and run the prep script, which:
1. sets up a venv
2. installs Ansible within that venv

```bash
./bin/prepare-venv
```

### Tune playbook behavior

Optionally configure `./src/extra_vars/custom-local.yml` to modify behavior:
```yaml
---
hey_you_read_this: |
  Uncomment any of the variables below and modify as desired. Doing so
  will override the role defaults and change the behavior of e.g. sshd
  or the firewall.

##############################################################################
#
#firewall_blackhole_timeout: "8h"
#sshd_allowed_group: "sshpass"
#sshd_port: 11066
#sshd_banner: |
#  ...........................................................................
#  Unauthorized use of this system and its networking resources is prohibited.
#
#           Violators will prosecuted to the full extent of the law.
#  ...........................................................................
#
##############################################################################
```

----------

## Bastion host deployment

**Important**: after running the Ansible playbook as specified below, do not disconnect from the system, or you will be locked out. Instead:
1. complete all the steps that follow in this deployment section
2. test a new SSH session without disconnecting from the current one

Only disconnect from the system after verifying that you can still SSH in.

### Run the Ansible playbook

Run the playbook helper script as root, or as an account that is a full sudoer:
```bash
./bin/run-playbook
```

### Configure authentication for shell account

Note that the ssh daemon for the bastion host requires *all of the following* in order to log in:
* successful pubkey authentication; and
* successful password authentication; and
* membership in the allowed SSH group

Install SSH public key for `someguy` shell account (and ensure correct permissions):
```bash
sudo cp someguy.pub /usr/local/etc/ssh/authorized_keys.someguy
```
```bash
sudo chmod 644 /usr/local/etc/ssh/authorized_keys.someguy
```

Set a password for `someguy` shell account:
```bash
sudo passwd someguy
```

Add `someguy` shell account to the allowed SSH group (note that this group will differ if you modified the `sshd_allowed_group` variable):
```bash
sudo usermod -aG sshpass someguy
```

Next, confirm `someguy` is able to successfully SSH in to the bastion host. If not, review `/var/log/secure` to diagnose.

----------

## Post-flight notes

Now that the bastion host is ready, there are several moving parts to be aware of.

| File                                     | Description                   |
| ---------------------------------------- | ------------------------------|
| `/etc/cron.d/ansible_auto-block-brutes`  | Periodic job that blocks the IP address of anyone who tries to SSH in as root or as an unknown account. [^block_baddies] The block is cleared after the value in variable `firewall_blackhole_timeout` passes by. |
| `/var/log/ansible-playbook`              | Log file containing entries from Ansible playbook execution. [^playbook_log] |
| `/var/log/bastion-chatter`               | Log file containing entries from custom script chatter, including that from periodic jobs. |

[^block_baddies]:
    The reasoning behind this IP blocking logic is:
    * Anyone who tries to SSH in as root is likely a bad actor
    * Anyone who tries to SSH in as an unknown account is likely a bad actor (i.e. conducting a brute force attempt with many different account names)

[^playbook_log]:
    Note that the first time you run the playbook, the log file will be empty (i.e. because it will not have been configured yet). For every successive playbook execution it will contain Ansible log entries.

If you intend to maintain the bastion host for an extended period of time, it may be a good idea to run this repo's Ansible playbook on some periodic schedule, e.g. in an example `/etc/cron.d/run-playbook`:

```
# Run bastion host playbook to ensure valid service configuration
_d='/local/path/to/this/repo'
30 05 * * * root cd "${_d}" && ./bin/run-playbook >/dev/null 2>&1
```
