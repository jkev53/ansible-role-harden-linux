ansible-role-harden-linux
=========================

This Ansible role was mainly created for [Kubernetes the not so hard way with Ansible (at Scaleway) - Part 2 - Harden the instances](https://www.tauceti.blog/post/kubernetes-the-not-so-hard-way-with-ansible-at-scaleway-part-2/). But it can be used also standalone of course to harden Linux (targeting Ubuntu 16.04/18.04 mainly at the moment). It has the following features:

- Change root password
- Add a regular/deploy user used for administration (e.g. for Ansible or login via SSH)
- Adjust APT update intervals
- Setup ufw firewall and allow only SSH access by default (add more rules/allowed networks if you like)
- Adjust security related sysctl settings (/proc filesystem)
- Change SSH default port (if requested)
- Disable password authentication
- Disable root login
- Disable PermitTunnel
- Install Sshguard and adjust whitelist

Versions
--------

I tag every release and try to stay with [semantic versioning](http://semver.org). If you want to use the role I recommend to checkout the latest tag. The master branch is basically development while the tags mark stable releases. But in general I try to keep master in good shape too.

Changelog
---------

**v3.0.0**

- Ansible v2.5 needed for Ubuntu 18.04 Bionic Beaver as Python 3 is default there. It *should* work with Ansible >= 2.2 too but who knows ;-) As Ubuntu 18.04 comes with Python 3 support only by default you may adjust your Ansible's `hosts` file. E.g you can add the `ansible_python_interpreter` env. like so: `host.domain.tld ansible_python_interpreter=/usr/bin/python3` (also see http://docs.ansible.com/ansible/latest/reference_appendices/python_3_support.html for more examples)

**v2.1.0**

- support for Ubuntu 18.04 Bionic Beaver
- added `sudo` package to `harden_linux_required_packages`

**v2.0.1**

- fixed deprecation warning while installing aptitude

**v2.0.0**

- major refactoring
- removed `common_ssh_port` (see `harden_linux_sshd_settings` instead)
- all variables that started with `common_` are now starting with the prefix `harden_linux_`. Additionally ALL variables that the role uses are now prefixed with `harden_linux_`. Using a variable name prefix avoids potential collisions with other role/group variables.
- introduced `harden_linux_deploy_user_uid` and `harden_linux_deploy_user_shell`
- single settings in `harden_linux_sysctl_settings` can be overriden by specifing the key/value in `harden_linux_sysctl_settings_user` list (whole list needed to be replaced before this change)
- more documentation added to `defaults/main.yml` (please read it ;-) )
- every setting in hosts `/etc/ssh/sshd_config` config file can now be replaced by using `harden_linux_sshd_settings_user` list. The defaults are specified in `harden_linux_sysctl_settings` and will be merged with `harden_linux_sysctl_settings_user` during run time.
- added variable `harden_linux_sshguard_whitelist` for Sshguard whitelist
- firewall rules can now be added using `harden_linux_ufw_rules` variable

**v1.0.0**

- initial release

Role Variables
--------------

The following variables don't have defaults. You need to specify them either in a file in `group_vars` or `host_vars` directory. E.g. if this settings should be used only for one specific host create a file for that host called like the FQDN of that host (e.g `host_vars/your-server.example.tld`) and put the variables with the correct values there. If you want to apply this variables to a host group create a file `group_vars/your-group.yml` e.g. Replace `your-group` with the host group name which you created in the Ansible `hosts` file (do not confuse with /etc/hosts...). `harden_linux_deploy_user_public_keys` loads all the public SSH key files specifed in the list from your local hard disk. So at least you need to specify:

```
harden_linux_root_password: your_encrypted_password_here

harden_linux_deploy_user: deploy
harden_linux_deploy_user_password: your_encrypted_password_here
harden_linux_deploy_user_home: /home/deploy
harden_linux_deploy_user_public_keys:
  - /home/your_user/.ssh/id_rsa.pub
```

With `harden_linux_root_password` and `harden_linux_deploy_user_password` we specify the password for the `root` user and the `deploy` user. Ansible won't encrypt the password for you. To create a encrypted password you can do so e.g. with `python -c 'import crypt; print crypt.crypt("This is my Password", "$1$SomeSalt$")'` (You may need `python2` instead of `python` in case of Archlinux e.g.).

`harden_linux_deploy_user` specifies the user we want to use to login at the remote host. As already mentioned the `harden-linux` role will disable root user login via SSH for a good reason. So we need a different user. This user will get "sudo" permission which we need for Ansible (and yourself of course) to do it's work.

`harden_linux_deploy_user_public_keys` specifies a list of public SSH key files you want to add to `$HOME/.ssh/authorized_keys` of the deploy user on the remote host. If you specify `/home/deploy/.ssh/id_rsa.pub` e.g. as a value here the content of that **local** file will be added to `$HOME/.ssh/authorized_keys` of the deploy user on the remote host.

The following variables below have defaults. So you only need to change them if you need another value for the variable. `harden_linux_required_packages` specifies the packages this playbook requires to work. You can extend the list but don't remove the packages listed:
```
harden_linux_required_packages:
  - ufw
  - sshguard
  - unattended-upgrades
  - sudo
```

The role changes some SSHd settings by default:
```
harden_linux_sshd_settings:
  "^PasswordAuthentication": "PasswordAuthentication no"  # Disable password authentication
  "^PermitRootLogin": "PermitRootLogin no"                # Disable SSH root login
  "^PermitTunnel": "PermitTunnel no"                      # Disable tun(4) device forwarding
  "^Port ": "Port 22"                                     # Set SSHd port
```
Personally I always change the default SSH port as lot of brute force attacks taking place against this port. So if you want to change the port setting for example you can do so:
```
harden_linux_sshd_settings_user:
  "^Port ": "Port 22222"
```
(Please notice the whitespace after "Port"!). The playbook will combine `harden_linux_sshd_settings` and `harden_linux_sshd_settings_user` while the settings in `harden_linux_sshd_settings_user` have preference which means it will override the `^Port ` setting/key in `harden_linux_sshd_settings`. As you may have noticed all the key's in `harden_linux_sshd_settings` and `harden_linux_sshd_settings_user` begin with `^`. That's because it is a regular expression (regex). One of playbook task's will search for a line in `/etc/ssh/sshd_config` e.g. `^Port ` (while the `^` means "a line starting with ...") and replaces the line (if found) with e.g `Port 22222`. This way makes the playbook very flexible for adjusting settings in `sshd_config` (you can basically replace every setting). You'll see this pattern for other tasks too so everything mentioned here holds true in such cases.

Next we have some defaults for our firewall/iptables:

```
harden_linux_ufw_defaults:
  "^IPV6": 'IPV6=yes'
  "^DEFAULT_INPUT_POLICY": 'DEFAULT_INPUT_POLICY="DROP"'
  "^DEFAULT_OUTPUT_POLICY": 'DEFAULT_OUTPUT_POLICY="ACCEPT"'
  "^DEFAULT_FORWARD_POLICY": 'DEFAULT_FORWARD_POLICY="DROP"'
  "^DEFAULT_APPLICATION_POLICY": 'DEFAULT_APPLICATION_POLICY="SKIP"'
  "^MANAGE_BUILTINS": 'MANAGE_BUILTINS=no'
  "^IPT_SYSCTL": 'IPT_SYSCTL=/etc/ufw/sysctl.conf'
  "^IPT_MODULES": 'IPT_MODULES="nf_conntrack_ftp nf_nat_ftp nf_conntrack_netbios_ns"'
```
This settings are basically changing the values in `/etc/defaults/ufw`. To override one or more of the default settings you can so by specifying the same key (which is a regex) as above e.g. `^DEFAULT_FORWARD_POLICY` and simply assign it the new value:

```
harden_linux_ufw_defaults_user:
  "^DEFAULT_FORWARD_POLICY": 'DEFAULT_FORWARD_POLICY="ACCEPT"'
```
As already mentioned above this playbook will also combine `harden_linux_ufw_defaults` and `harden_linux_ufw_defaults_user` while the settings in `harden_linux_ufw_defaults_user` have preference.

Next we can specify some firewall rules with `harden_linux_ufw_rules`. This is the default:

```
harden_linux_ufw_rules:
  - rule: "allow"
    to_port: "22"
    protocol: "tcp"
```
You can add more settings for a rule like `interface`, `from_ip`, ... Please have a look at `tasks/main.yml` (search for "Apply firewall rules") for all possible settings.

You can also allow hosts to communicate on specific networks (without port restrictions) e.g.:
```
harden_linux_ufw_allow_networks:
  - "10.3.0.0/24"
  - "10.200.0.0/16"
```

Next harden-linux role also changes some system variables (sysctl.conf / proc filesystem). This settings are recommendations from Google which they use for their Google Compute Cloud OS images (see https://cloud.google.com/compute/docs/images/building-custom-os and https://cloud.google.com/compute/docs/images/configuring-imported-images). This are the default settings (if you are happy with this settings you don't have to do anything but I recommend to verify if they work for your setup):

```
harden_linux_sysctl_settings:
  "net.ipv4.tcp_syncookies": 1                    # Enable syn flood protection
  "net.ipv4.conf.all.accept_source_route": 0      # Ignore source-routed packets
  "net.ipv6.conf.all.accept_source_route": 0      # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_source_route": 0  # Ignore source-routed packets
  "net.ipv6.conf.default.accept_source_route": 0  # IPv6 - Ignore source-routed packets
  "net.ipv4.conf.all.accept_redirects": 0         # Ignore ICMP redirects
  "net.ipv6.conf.all.accept_redirects": 0         # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_redirects": 0     # Ignore ICMP redirects
  "net.ipv6.conf.default.accept_redirects": 0     # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.all.secure_redirects": 1         # Ignore ICMP redirects from non-GW hosts
  "net.ipv4.conf.default.secure_redirects": 1     # Ignore ICMP redirects from non-GW hosts
  "net.ipv4.ip_forward": 0                        # Do not allow traffic between networks or act as a router
  "net.ipv6.conf.all.forwarding": 0               # IPv6 - Do not allow traffic between networks or act as a router
  "net.ipv4.conf.all.send_redirects": 0           # Don't allow traffic between networks or act as a router
  "net.ipv4.conf.default.send_redirects": 0       # Don't allow traffic between networks or act as a router
  "net.ipv4.conf.all.rp_filter": 1                # Reverse path filtering - IP spoofing protection
  "net.ipv4.conf.default.rp_filter": 1            # Reverse path filtering - IP spoofing protection
  "net.ipv4.icmp_echo_ignore_broadcasts": 1       # Ignore ICMP broadcasts to avoid participating in Smurf attacks
  "net.ipv4.icmp_ignore_bogus_error_responses": 1 # Ignore bad ICMP errors
  "net.ipv4.icmp_echo_ignore_all": 0              # Ignore bad ICMP errors
  "net.ipv4.conf.all.log_martians": 1             # Log spoofed, source-routed, and redirect packets
  "net.ipv4.conf.default.log_martians": 1         # Log spoofed, source-routed, and redirect packets
  "net.ipv4.tcp_rfc1337": 1                       # Implement RFC 1337 fix
  "kernel.randomize_va_space": 2                  # Randomize addresses of mmap base, heap, stack and VDSO page
  "fs.protected_hardlinks": 1                     # Provide protection from ToCToU races
  "fs.protected_symlinks": 1                      # Provide protection from ToCToU races
  "kernel.kptr_restrict": 1                       # Make locating kernel addresses more difficult
  "kernel.perf_event_paranoid": 2                 # Set perf only available to root
```
You can override every single setting e.g. by creating a variable called `harden_linux_sysctl_settings_user`:
```
harden_linux_sysctl_settings_user:
  "net.ipv4.ip_forward": 1
  "net.ipv6.conf.default.forwarding": 1
  "net.ipv6.conf.all.forwarding": 1
```
One of the playbook tasks will combine `harden_linux_sysctl_settings` and `harden_linux_sysctl_settings_user` while again `harden_linux_sysctl_settings_user` settings have preference. Have a look at `defaults/main.yml` file of the role for more information about the settings.

If you want UFW logging enabled set:
```
harden_linux_ufw_logging: 'on'
```
Possible values are `on`,`off`,`low`,`medium`,`high` and `full`.

And finally we've the Sshguard settings. Sshguard protects from brute force attacks against SSH. To avoid locking out yourself for a while you can add IPs or IP ranges to a whitelist. By default it's basically only "localhost":
```
harden_linux_sshguard_whitelist:
  - "127.0.0.0/8"
  - "::1/128"
```

Example Playbook
----------------

If you installed the role via `ansible-galaxy install githubixx.harden-linux` then include the role into your playbook like in this example:

```
- hosts: webservers
  roles:
    - githubixx.harden-linux
```

License
-------

GNU GENERAL PUBLIC LICENSE Version 3

Author Information
------------------

[www.tauceti.blog](https://www.tauceti.blog)
