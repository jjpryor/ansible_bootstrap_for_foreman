# bootstrap_for_foreman
Ansible playbooks for installing Foreman 1.9 on RHEL7.x on target nodes,
a Fedora 22 for desktop machine to run ansible, and Ansible 1.9+.

This set of playbooks is composed from the ["Foreman 1.9 Manual"]
(http://www.theforeman.org/manuals/1.9/index.html) instructions.

# Start here with remote account pre-reqs
git clone this repo to the a working area in your `$HOME`.

On the foreman node, perform the following steps to enable the remote user and
the remote user's public SSH key placed into the `.ssh/authorized_keys` file.

SSH into the target node:
 ```shell
 ssh root@targetnode
 ```

Now as root, add the nonprivuser:
```shell
useradd nonprivuser -u SomeLargeNumberToPreventUidCollision
```

Add `nonprivuser` to /etc/group `wheel`: `usermod -G wheel -a nonprivuser`

Change the `sudoers` file to have the wheel group able to run sudo commands
without requiring a password. *This is obviously weakens security, and should
be removed later*
```
%wheel  ALL=(ALL)       NOPASSWD: ALL
```

Now become the `nonprivuser`:
```shell
su - nonprivuser
```

As nonprivuser, make the .ssh directory, set the correct permissions, and
copy/paste the `authorized_keys` file into place, then exit the `nonprivuser`:
```shell
cd
mkdir .ssh
chmod 600 .ssh
vim .ssh/authorized_keys
<paste the public key>
chmod 600 .ssh/authorized_keys
exit
```

From your desktop, enable `ssh-agent` if your desktop environment doesn't
automatically have it running, and also add your ssh identity to the agent.

(*Quite possibly the wrong way to do this, but it works*)
```shell
exec ssh-agent bash
ssh-add ~/.ssh/your_private_key
```
Then check if ssh nonprivuser@targetnode via key & passphrase works and also
test if password-less sudo works:
```shell
ssh -i ~/.ssh/your_private_key nonprivuser@targetnode
sudo touch /etc/foobar
```

Now test ansible to see if it will work:
```
ansible -m setup foreman-node.yourorg.local
```
It should connect via SSH and gather facts of the target
and display the facts in the console, something like:
```
foreman-node.yourorg.local | success >> {
    "ansible_facts": {
        "ansible_all_ipv4_addresses": [
            "111.222.323.423"
        ],
        "ansible_all_ipv6_addresses": [
            "fe80::250:6bff:fe89:689f"
        ],
        "ansible_architecture": "x86_64",
        "ansible_bios_date": "04/14/2014",
        "ansible_bios_version": "6.00",
        "ansible_cmdline": {
            "BOOT_IMAGE": "/vmlinuz-3.10.0-229.el7.x86_64",
            "crashkernel": "auto",
            "nofb": true,
            "rd.lvm.lv": "sysvg/root",
            "ro": true,
            "root": "/dev/mapper/sysvg-root"
        },
```
## Hardware & Disk space requirements
Just about ready to run the ansible playbooks, but we need to check
if there is not enough space in the filesystem, then use `lvresize` to resize
logical volume of the partition.
Examples:
```shell
lvresize -r -L 110G /dev/mapper/optvg-opt
lvresize -r -L 45G /dev/mapper/varvg-var
lvresize -r -L 5G /dev/mapper/sysvg-tmp
```

| Node | Cores | RAM | /opt/ | /var/ | /tmp/ |
| --- | --- |  --- |  --- |  --- |  --- |
|Foreman node|2|8 GB|2 GB|8 GB| 4G free|

# Execute the ansible playbooks
If it is all successful, then it is time to start using the ansible playbooks.
They are designed to get all the pieces in place and check to see if the disk
space requirements are met.

```Shell
ansible-playbook 01-prereqs.yml
ansible-playbook 02-bootstrap-foreman.yml
```
At this point in time it will not run the installer for you.

---

## Contents of split-node-pe directory
### ansible.cfg
I'm pointing ansible to use my SSH private key and I'm using my `jjpryortemp` remote account that must be manually setup on each of the three nodes.

### ansible_hosts
It should be obvious that this is where you define the host that these playbooks
will be interacting with.
```
[foreman-node]
foreman-node.yourorg.local
```

### 01-prereqs.yml
+ Performs `yum update -y`
+ Enables `$proxy_*tp` bash shell environment variables
+ Disables & stops `firewalld`
+ Enables & starts `iptables`
+ Checks `/opt` & `/var` & `/tmp` for minimum size for Foreman.

### 02-bootstrap-foreman.yml
+ Sets the following environment variables:
  + `ftp_proxy`
  + `http_proxy`
  + `https_proxy`
  + `no_proxy`
+ Prompts the user for the RHN username and password
+ Adds the "Software Collections" YUM repo to the machine
+ Downloads and installs the required RPMS
+ Installs the `foreman-installer` program.

-----
## Contents of files directory

### files/proxy.sh
Shell environment variables for the proxy & this goes into `/etc/profile.d`

### files/wgetrc
wget's config file & this goes into `/etc/`

### files/iptables
iptables config for Foreman

### epel.repo
The EPEL YUM repo file that points to the local mirror.
