# STIGd_AAP_Install
Instructions for how to install Ansible Automation Platform Containerized 2.6/2.7 on RHEL 9/10 that has been STIG'd.

Below is my standard guide to doing Containerized AAP 2.6/2.7 installs on RHEL 9/10. This is provided with no warranty expressed or implied, these are just instructions found to usually work for hardened environments.

## Pre-Reading

Here is a link to the hardening guide for Ansible Automation Platform 2.6:
https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/secure-assembly_hardening_aap

And the same guide for AAP 2.7:
https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.7/secure-assembly_hardening_aap

## Binaries

RHEL ISOs are located here: https://access.redhat.com/downloads/content/rhel

If you are doing a disconnected install, you need to get the "Red Hat Enterprise Linux 9.x Binary DVD" (or 10.x) for this, if you are connected to the internet or a Satellite server, then you can get the "Red Hat Enterprise Linux 9.x Boot ISO" (or 10.x)

The AAP 2.6 binaries are available here: https://access.redhat.com/downloads/content/480/ver=2.6/rhel---9/2.6/x86_64/product-software

If you need a different combination, change the selector accordingly (2.6/2.7 and/or RHEL 9/10).

You need the "Ansible Automation Platform 2.x Containerized Setup Bundle" for an offline install, otherwise you can use the smaller "Ansible Automation Platform 2.x Containerized Setup"

## Partitions

I typically allocate a disk with 200GB due to containers and bundled installs in the mix, the extracted containers take up more space. All of these numbers can be increased if you so desire, and you could leave some space unused on a larger drive if you are concerned about changing partition sizes later. Since the STIG requires /home to be mounted noexec (V-257852), you will need to install AAP in the /opt directory. A fresh baseline install of AAP uses ~23GB of space, so if you are concerned and have more storage available, make things bigger:

```
/boot            1GB
/boot/efi        1GB
/tmp            20GB
/var            20GB
/var/tmp        20GB
/var/log        10GB
/var/log/audit   5GB
/home           10GB per user
swap             8GB
/opt            60GB 
/          Remaining
```

Partitions are set up during installation of RHEL.

## FAPolicy

STIGs should always be applied as early as possible and before AAP is installed. During installation of RHEL 9, you select the security profile and select the DISA STIG for RHEL profile.  For RHEL 10, this option is no longer available during boot. You will have to apply DISA STIG policies after initial boot. Either way, fapolicyd (File Access Policy) will be enforcing, and its default policy blocks the AAP installer.

Why it blocks: the STIG keeps RHEL's stock fapolicyd rules in /etc/fapolicyd/rules.d/ and adds a final `99-deny-everything.rules`. The stock rule that matters is `deny_audit perm=any all : ftype=%languages` (in `70-trusted-lang.rules`). It denies opening any **untrusted** Python (or other script-language) file, whatever program opens it and wherever the file is. The install involves two kinds of untrusted Python:

1. **The temporary module files Ansible writes for each task** (`AnsiballZ_*.py`). Pipelining fixes this; see "Modify ansible.cfg" below.
2. **The installer's own code in the bundle's `collections/` directory.** `ansible-playbook` runs as the install user and has to read these modules and plugins. Pipelining doesn't help with this part.

Pick one of these two approaches with your ISSM:

**Option A: permissive for the install window.** This is what Red Hat documents in the AAP 2.7 hardening guide. Before installing:
```
sudo sed -i 's/^\s*permissive\s*=.*/permissive = 1/' /etc/fapolicyd/fapolicyd.conf
sudo systemctl restart fapolicyd
sleep 5; systemctl is-active fapolicyd
```
Set it back to `permissive = 0` after the install (see Post-Installation).

**Option B: stay enforcing, and trust the installer's files for the install window.** This adds no fapolicyd rules. The commands are in "fapolicyd: trust the installer (Option B)" under the installation steps below. They have to run after the installer path fixes, because the `sed` step edits the collection files.

Either way, check `systemctl is-active fapolicyd` a few seconds after any fapolicyd restart. A restart can look successful while the daemon exits during startup, and a stopped fapolicyd enforces nothing.

Do **not** write `allow ... uid=<ANSIBLE_USER> : dir=...` rules for a directory the install user can write, such as `/opt/ansible`. Those rules have no `trust=` condition, so the account could then run any binary or script it places in that directory, and fapolicyd would no longer apply to the account that runs AAP.

You can learn more about FA Policy here: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening

## User Namespaces

Next, we need to allow containers to run on RHEL. A STIG denies this by default but it allows for containers if deemed necessary. It is necessary. Note the following sentence in the STIG "If the use of namespaces is operationally required and documented with the ISSM, it is not a finding." - https://stigviewer.com/stigs/red_hat_enterprise_linux_9/2025-02-27/finding/V-257816

Fix this with the following:

```
sudo sysctl user.max_user_namespaces=15000
```

The above will make the change initially. To have this setting persist, run the following command:

```
echo "user.max_user_namespaces=15000" | sudo tee /etc/sysctl.d/99-user-namespaces.conf
```

Note: 15000 is a relatively arbitrary number. After installation and steady state running in production, you can tune this number down to something more reasonable. To find out how many namespaces are in use, you can run the following command as the user that Ansible Automation Platform is running under:

```
lsns -t user -n | wc -l
```
(Run it without `sudo`. With `sudo` it counts every user's namespaces on the system, plus the header line.)

Be sure to give yourself plenty of head room, perhaps double the number shown by that command to accommodate Execution Environments running and multiple users on the system.

## Other STIG settings that affect the install

- **Audit backlog (can crash the host).** The STIG sets `audit_backlog_limit=8192` together with `-f 2`, which panics the kernel when an audit record is lost. Rootless Podman unpacking and starting the AAP containers generates audit records faster than auditd writes them. In testing, 8192 panicked the kernel during the install. Raise the limit in both places, keep `-f 2`, and reboot (the audit rules are immutable until reboot). The STIG's SCAP rule still passes with 65536:
  ```
  sudo grubby --update-kernel=ALL --args=audit_backlog_limit=65536
  sudo sed -i 's/^\s*-b\s\+8192/-b 65536/' /etc/audit/rules.d/audit.rules
  ```
  With 65536, no audit records were lost in any of our installs. But the queue came within 1% of the limit while images were loading and at boot, and with 131072 it filled completely. A full queue makes processes wait rather than drop records, but a record that can't wait would still be lost and panic the host. Keep AIDE scans and other audit-heavy work out of the install window, and discuss the install-window risk with your ISSM.
- **AIDE.** The STIG schedules a daily AIDE scan. Scanning rootless container storage adds audit load and reports constant changes. Exclude the container storage and the AAP data directory in `/etc/aide.conf` (use the real paths, which are under /opt/ansible with this layout), then re-initialize the AIDE database after the install.
- **umask.** The STIG sets 0077. Red Hat's 2.7 docs require 0022 or 0002 for the install; with 077 the controller task container can fail to start. Run the installer from a shell with `umask 0022`.
- **sudo.** The STIG deletes NOPASSWD sudo rules entirely. Give the install user password-based sudo (for example membership in `wheel`) before hardening, and supply the become password to the installer.

## Installation Location

IF you are installing in a location other than /home (and you should be to adhere to STIGs), we have to change some hardcoded paths within the installer so it knows to place things. The standard path is /home/<username>, but the STIG requires /home to be mounted noexec (V-257852), so nothing under /home can be executed. That breaks a default install: the installer bind-mounts `~/aap/controller/etc/launch_awx_task.sh` into the controller containers as their entrypoint, and on a noexec /home the controller task container fails to start (`/usr/bin/launch_awx_task.sh: Permission denied`). It does the same with `~/aap/containers/podman`, which receptor uses to launch job containers. Follow the instructions in this solution, which uses /opt as an installation location with a username of ansible. If you are using another dedicated user for the installation, use that instead:

https://access.redhat.com/solutions/7117294 

## Installation Instructions

Finally, the actual instructions for installation are located here: https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/aap-containerized-installation#installing-containerized-aap 

Here are the steps contained in those instructions:

### Folders

Create ansible base directory with root user
In this case use the /opt/ansible for example, you can specify it with any name but /opt is a generally acceptable directory for executables and ansible is the name of the user created to run AAP.
Do all the following steps as the ansible user, or the user you have created to run AAP.

```
sudo mkdir -p /opt/ansible
sudo mkdir -p /opt/ansible/.config
sudo mkdir -p /opt/ansible/.local
sudo chown ansible:ansible  -R /opt/ansible
```

### Cleanup

If AAP was already installed, remove it before proceeding.

```
rm -rfv ~/aap
rm -rfv ~/.config
rm -rfv ~/.local
```

### Environment variables

Customize the XDG path with soft link.

```
ln -s /opt/ansible/.config ~/.config
ln -s /opt/ansible/.local ~/.local 
```

Optionally, but recommended:

```
echo "export XDG_RUNTIME_DIR=/run/user/$(id -u)" >> ~/.bashrc
echo "export XDG_CONFIG_HOME=/opt/ansible/.config" >> ~/.bashrc
echo "export XDG_DATA_HOME=/opt/ansible/.local/share" >> ~/.bashrc
```

Note: the above commands should be executed on all the AAP nodes.

### Fix installer path locations

Use the customized path variable to replace the default path variable in the installation path.

```
cd <unpack_installation_path>/collections
## replace {{ ansible_user_dir }}/aap with {{ ansible_base_dir }}/aap
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_user_dir }}/aap#{{ ansible_base_dir }}/aap#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/aap#{{ ansible_base_dir }}/aap#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.local#{{ ansible_local_dir }}/.local#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_user_dir }}/.local#{{ ansible_local_dir }}/.local#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.config#{{ ansible_config_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_user_dir }}/.config#{{ ansible_config_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.gnupg#{{ ansible_config_dir }}/.gnupg#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}#{{ ansible_base_dir }}#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_config_dir }}/.config#{{ ansible_user_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_config_dir }}/.config#{{ ansible_user_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_local_dir }}/.local#{{ ansible_user_dir }}/.local#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_local_dir }}/.local#{{ ansible_user_dir }}/.local#g' {} +
```

With fapolicyd enforcing (Option B), expect `sed: can't read …/lightspeed_settings.py.j2: Operation not permitted`, and the same for one community.crypto test file. fapolicyd classifies both as Python. Neither contains the paths being changed, so the messages are harmless.

### Modify Inventory

Add the following variables to the install inventory

```
[all:vars]
ansible_base_dir=/opt/ansible
ansible_config_dir=/opt/ansible
ansible_local_dir=/opt/ansible
```

### Modify ansible.cfg

When Ansible runs normal playbooks and tasks, it deploys a small amount of python code on the end server, which then executes the necessary commands. Since this is compiled essentially at runtime, it cannot be trusted before installation and cannot be baked into FAPolicy. Therefore, we must take advantage of an Ansible feature called pipelining, which will stream the commands via stdin instead of using binaries. Pipelining is required for fapolicyd Option B, and harmless with Option A. It does not cover the installer's own code in `collections/`; that's what the trust step below is for. Customized the ansible.cfg file in your installer folder with the following settings:

```
[defaults]
collections_on_ansible_version_mismatch = ignore
collections_path = ./collections
inventory = ./inventory
log_path = ./aap_install.log
remote_tmp = /opt/ansible/tmp
local_tmp = /opt/ansible/tmp

[connection]
pipelining = True
```


### fapolicyd: trust the installer (Option B)

Skip this if you chose Option A. Run it after "Fix installer path locations" and "Modify ansible.cfg", and before the install. Make the installer's collections read-only to the install user, then trust exactly those files:

```
cd /opt/ansible/<unbundled-installer-directory>
sudo chown -R root:root collections
sudo chmod -R u=rwX,go=rX collections
sudo fapolicyd-cli --file add "$PWD/collections/" --trust-file aap-installer
sudo fapolicyd-cli --update
sleep 5; systemctl is-active fapolicyd
```

"Skipping symbolic link" messages from `--file add` are expected. Don't skip the `chown`. The STIG leaves fapolicyd's integrity checking off (`integrity = none`), so trust is by path, and root ownership is what stops the trusted files from being changed. The `chmod` keeps the tree readable, but not writable, by the install user. That matters because a bundle extracted on the hardened host (umask 0077) starts out owner-only, and once root owns it the install user couldn't read it.

Check that the trust took effect:

```
ansible localhost -m debug -a "msg={{ '::1' | ansible.containerized_installer.ipv6_wrap }}"
```

You should see `"msg": "[::1]"`. If you see `Operation not permitted`, the trust isn't active.

Repeat this (and remove it afterwards) for every installer run: reruns, upgrades (trust the new bundle's `collections/`), backup, restore, and uninstall. They all load the same code.

### Install

Ensure the hostname is set

```
hostname -f
```

Ensure that Ansible Core and python3-requests are installed (the 2.7 installer needs python3-requests)

```
sudo dnf install -y ansible-core python3-requests
```

Optionally install some troubleshooting repositories

```
sudo dnf install -y wget git-core rsync vim
```

Install AAP 2.x Containerized from /opt/ansible/unbundled-installer-directory/

```
ansible-playbook  -i inventory ansible.containerized_installer.install
```

On a STIG'd host, also account for these (see "Other STIG settings that affect the install"):
- Run the installer with `umask 0022`.
- Provide the become password, either with `--ask-become-pass` or from a protected vars file with `-e @file`.
- Run it detached (for example with `nohup ... &`). The install takes about an hour, and the STIG stops idle sessions after 10 minutes.

## Post-Installation

Return fapolicyd to its normal state, then reboot and confirm there are no denials:
- **Option A:** `sudo sed -i 's/^\s*permissive\s*=.*/permissive = 0/' /etc/fapolicyd/fapolicyd.conf`
- **Option B:** `sudo fapolicyd-cli --file delete "$PWD/collections/" --trust-file aap-installer && sudo fapolicyd-cli --update`, run from the installer directory.

```
sudo systemctl reboot
# after the host is back:
systemctl is-active fapolicyd
sudo ausearch -m FANOTIFY -ts boot      # expect no denials; the running platform needs no fapolicyd exception
```

At this point, you can begin to configure third party authentication, git integration, RBAC, and other features.
