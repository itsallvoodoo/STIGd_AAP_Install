# STIGd_AAP_Install
Instructions for how to install AAP 2.6 on RHEL 9 that has been STIG'd.

Below is my standard guide to doing Containerized AAP 2.6 installs on RHEL 9. This is provided with no warranty expressed or implied, these are just instructions found to work for hardened environments.

## Pre-Reading

Here is a link to the hardening guide for Ansible Automation Platform 2.6:
https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/pdf/hardening_and_compliance/Red_Hat_Ansible_Automation_Platform-2.6-Hardening_and_compliance-en-US.pdf

## Binaries

RHEL ISOs are located here: https://access.redhat.com/downloads/content/rhel

If you are doing a disconnected install, you need to get the "Red Hat Enterprise Linux 9.6 Binary DVD" for this, if you are connected to the internet or a Satellite server, then you can get the "Red Hat Enterprise Linux 9.6 Boot ISO"

The AAP binaries are available here: https://access.redhat.com/downloads/content/480/ver=2.6/rhel---10/2.6/x86_64/product-software

You need the "Ansible Automation Platform 2.6 Containerized Setup Bundle" for an offline install, otherwise you can use the smaller "Ansible Automation Platform 2.6 Containerized Setup"

## Partitions

I typically allocate a disk with 200GB due to containers and bundled installs in the mix, the extracted containers take up more space. All of these numbers can be increased if you so desire, and you could leave some space unused on a larger drive if you are concerned about changing partition sizes later. Since STIGs forbids the installation of applications in /home directories, you will need to install AAP in the /opt directory, and that is why / is the largest. If you are concerned and have more storage available, make things bigger:

```
/boot.           1GB
/boot/efi        1GB
/              120GB
/home           10GB per user
/tmp            20GB
/var/tmp        20GB
/var/log        10GB
/var/log/audit   5GB
```

Partitions are set up during installation of RHEL.

## FAPolicy

Also during installation, you select the security profile and select the DISA STIG for RHEL profile. This will enforce a lot of the security requirements needed to be compliant. We must configure fapolicy rules (File Access Policy) to allow Ansible to run.

The rules file needs to be 15-custom.rules inside the /etc/fapolicyd/rules.d/ folder, and includes the following. Copy and paste the below into your "15-custom.rules" file created.

```
%languages=application/x-bytecode.ocaml,application/x-bytecode.python,application/java-archive,text/x-java,application/x-java-applet,application/javascript,text/javascript,text/x-awk,text/x-gawk,text/x-lisp,application/x-elc,text/x-lua,text/x-m4,text/x-nftables,text/x-perl,text/x-php,text/x-python,text/x-R,text/x-ruby,text/x-script.guile,text/x-tcl,text/x-luatex,text/x-systemtap
allow perm=execute uid=<ANSIBLE_USER> : dir=/opt/.ansible
allow perm=open uid=<ANSIBLE_USER> : dir=/opt/.ansible
allow perm=any uid=0 : dir=/var/tmp/
allow perm=any uid=0 trust=1 : all
allow perm=open exe=/usr/bin/rpm : all
allow perm=open exe=/usr/bin/<PYTHONBINARYVERSION> comm=dnf : all
deny_audit perm=any pattern=ld_so : all
deny_audit perm=any all : ftype=application/x-bad-elf
allow perm=open all : ftype=application/x-sharedlib trust=1
deny_audit perm=open all : ftype=application/x-sharedlib
allow perm=execute all : trust=1
allow perm=open all : ftype=%languages trust=1
deny_audit perm=any all : ftype=%languages
allow perm=any all : ftype=text/x-shellscript
deny_audit perm=execute all : all
allow perm=open all : all
```

Be sure to change where it says \<ANSIBLE_USER\> . That needs to be set to the user ID of the account that will be installing AAP. You can determine that 4 digit number by running the command "id user" with whatever user name you are going to use. Below is an example of this:
```
[~]$ id alex
```
uid=1000(alex) gid=1000(alex) groups=1000(alex),10(wheel)
1000 is the uid in this example.

Also change the \<PYTHONBINARYVERSION\> to be whichever python executable you have installed. This can sometimes be /usr/bin/python3, but is usually just /usr/bin/python .

Lastly on FAPolicy, if you have custom rules that already exist within your /etc/fapolicyd/rules.d/ folder, you may need to review them to ensure you don't have duplicate lines. The FAPolicy process with merge all rules files into a single combined rules file after you restart the service or the server. Run the following command:
```
sudo systemctl restart fapolicyd
```
If you run into an issue during restart, review the /etc/fapolicyd/compiled.rules file to see if there are overlaps or direct conflicts with existing rules.

You can learn more about FA Policy here: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/security_hardening/assembly_blocking-and-allowing-applications-using-fapolicyd_security-hardening

## User Namespaces

Next, we need to allow containers to run on RHEL. A STIG denies this by default but it allows for containers if deemed necessary. It is necessary. Note the following sentence in the STIG "If the use of namespaces is operationally required and documented with the ISSM, it is not a finding." - https://stigviewer.com/stigs/red_hat_enterprise_linux_9/2025-02-27/finding/V-257816

Fix this with the following:

```
sudo sysctl user.max_user_namespaces=15000
```

## Installation Location

IF you are installing in a location other than /home (and you should be to adhere to STIGs), we have to change some hardcoded paths within the installer so it knows to place things. The standard path is /home/<username>, but STIGs do not allow for executable binaries in /home. Follow the instructions in this solution, which uses /opt as an installation location with a username of ansible. If you are using another dedicated user for the installation, use that instead:

https://access.redhat.com/solutions/7117294 

## Installation Instructions

Finally, the actual instructions for installation are located here: https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html/containerized_installation/aap-containerized-installation#installing-containerized-aap 

Here are the steps contained in thsoe instructions:

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

### Fix intaller path locations

Use the customized path variable to replace the default path variable in the installation path.

```
cd <unpack_installation_path>/collections
## replace {{ ansible_user_dir }}/aap with {{ ansible_base_dir }}/aap
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_user_dir }}/aap#{{ ansible_base_dir }}/aap#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/aap#{{ ansible_base_dir }}/aap#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.local#{{ ansible_local_dir }}/.local#g' {} +
find ansible_collections -type f -name “*.j2” -exec sed -i 's#{{ ansible_user_dir }}/.local#{{ ansible_local_dir }}/.local#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.config#{{ ansible_config_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_user_dir }}/.config#{{ ansible_config_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}/.gnupg#{{ ansible_config_dir }}/.gnupg#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_user_dir }}#{{ ansible_base_dir }}#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_config_dir }}/.config#{{ ansible_user_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_config_dir }}/.config#{{ ansible_user_dir }}/.config#g' {} +
find ansible_collections -type f -name "*.yml" -exec sed -i 's#{{ ansible_local_dir }}/.local#{{ ansible_user_dir }}/.local#g' {} +
find ansible_collections -type f -name "*.j2" -exec sed -i 's#{{ ansible_local_dir }}/.local#{{ ansible_user_dir }}/.local#g' {} +
```

### Modify Inventory

Add the following variables to the install inventory

```
[all:vars]
ansible_base_dir=/opt/ansible
ansible_config_dir=/opt/ansible
ansible_local_dir=/opt/ansible
```

### Install

Install AAP 2.6 Containerized

```
ansible-playbook  -i inventory ansible.containerized_installer.install
```

## Post-Installation

At this point, you can begin to configure third party authentication, git integration, RBAC, and other features.
