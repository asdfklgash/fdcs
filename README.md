# FDCS - Froxlor Docker Customer Shell

## About

Here are some scripts that can be used to setup dockerized ssh environments for
[Froxlor](https://www.froxlor.org) customers.

## Why?

Because i needed a secure(r) ssh environment and Froxlor doesn't bringt that OOTB.

## Status

"Works for me". If you want to use it or get some ideas then be my guest. But expect some problems due to missing
documentation or bugs. Also consider security: This is probably safer than direct ssh access, but might not be suitable
for all situations, especially when you need a very high security level.

**If you don't feel comfortable with froxlor and linux in general this is probably not for you at this point.**

## How does it work?

- /usr/local/fdcs/bin/fdcs is added to the list of valid shells in Froxlor Panel
- a user can now select this shell for ftp users
- when the user logs in a docker container for this user is started using the
  pam_exec module. This still runs in root context, we we can easily start a
  docker container. The name of this container will be fdcs_<username>.
  The script uses [docker-customer-chroot](https://github.com/mmunz/docker-customer-chroot)
  as docker base image, which is a slim debian with some tools and php7.0-cli added. Other images will (probably) work
  as well.
- A custom home-directory is created for the user in /var/customers/home/<username> and the $HOME variable set to this
  path.
- OpenSSH is made to look for authorized keys in this custom home directory. So the user can place his ssh keys in the
  usual location (~/.ssh/authorized_keys)
- Directories the user needs to access are bind mounted with -v into the container. We also bind mount /etc/passwd and
  /etc/group using the files generated by libnss-extrausers.
- To make mysql from the froxlor host available inside the docker container first the mysql socket is bind mounted
  into the container. Then we use socat to make this socker available inside the container on 127.0.0.1:3306.
- /usr/local/fdcs/bin/fdcs (the users login shell) now uses docker-exec to start
  a bash shell or run commands in the container. Because only root and group members
  of docker can use docker commands we need to sudo to the user fdcs for this.
  Passwordless execution of the entry script is allowed for all fdcs users.
- Because every fdcs user must be a member of the fdcs group a cronjob needs to
  be setup to add members to the fdcs group.
- When the user logs out we wait some time and remove the container afterwards.

## Installation

- make sure docker ist installed and works.
- You need a recent kernel for cpu limit support. 3.16 from jessie is to old. I use 4.9.0 from jessie-backports.
- enable memory cgroups. On debian jessie (8) add the following to /etc/default/grub. Debian 9 has cgroups enabled
  already, so if using Debian 9, skip this step.

  ```
  GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
  ```
  
  After that update grub with *update-grub* and reboot.
- you can check with docker version if cpu and memory cgroups are available. If they aren't you will see a warning. 
- **make sure you use froxlor with libnss-extrausers instead of libnss-mysql.**
  We need the extrausers files to mount them inside the container.
  libnss-extrausers is only available in recent versions of Froxlor, you may even have to use master from git.
- clone this repository, e.g. to /usr/local/

  ```
  cd /usr/local
  git clone https://github.com/mmunz/fdcs.git
  ```
  
  **If you are using another path than /usr/local make sure to adapt all paths (e.g. in sudoers and pam files)**
- adapt the configuration to your environment, see **config/default.config.sh**
- add fdcs system user and add it to the docker group:

  ```
  adduser --system --gid 65534 --no-create-home --disabled-login fdcs
  addgroup fdcs docker
  ```
  This user doesn't have to be in the group fdcs, so we set the group to 65534 (nogroup)
- Setup a sudoers file to allow passwordless execution of the fdcs-entry script:

  ```
    echo "%fdcs ALL=(fdcs) NOPASSWD: /usr/local/fdcs/bin/fdcs-entry" > /etc/sudoers.d/fdcs
    chmod 440 /etc/sudoers.d/fdcs
  ```
- Make sure, that files in /etc/sudoers.d/ are really included. There should be a line (yes, including the comment
  before) like the following in /etc/sudoers
  
  ```
  #includedir /etc/sudoers.d
  ```
- setup pam_exec: Add a line like the following to /etc/pam.d/sshd:

  ```
  session    required     pam_exec.so seteuid /usr/local/fdcs/pam/fdcs-pam-sshuser
  ```
- Setup the OpenSSh server, add this to /etc/ssh/sshd_config:

  ```
  Match Group fdsc
  X11Forwarding no
  AllowTcpForwarding no
  AuthorizedKeysFile /var/customers/home/%u/.ssh/authorized_keys
  ```
  This is needed for ssh logins with ssh public keys. 
- Setup a cron task to add fdcs-shell users to the fdcs group. Put the following into /etc/cron.d/fdcs and restart cron.
  Chmod the file to 640. 
  ``` 
  # crontab for fdcs (froxlor docker customer shell)
  PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
  #
  */5 * * * * root /usr/bin/php /usr/local/fdcs/cron/fdcs-cron.php 1> /dev/null
  ```
  Don't forget to restart cron with */etc/init.d/cron restart* now.
- Configure Froxlor (in the Froxlor panel):
  - in *settings->security options* 
    - enable **Allow customers to enable shell access for ftp-users**,
    - add "`/usr/local/fdcs/bin/fdcs`" to **List of available shells** and
    - add "`fdcs`" to **Custom system group for all customer users** 
  - Switch to a user in the panel. For each FTP account you are now able to select fdcs as shell and use the users
    credentials to login using SSH (after some minutes). Add public keys to ~/.ssh/authorized_keys when logged in
    as user via SSH to enable public key authentication.

## Security

I tried to make this as safe as possible, e.g.:

- drop all privileges in the container and run as logged in user
- readonly root filesystem
- all setuid binaries were removed in the docker-debian-chroot image
- only the fdcs-entry script is allowed to be executed as fdcs user with sudo
- Each container runs with limited resources (cpu/ram/ulimits) and has its own network

If you find vulnerabilities or have some suggestions how to make this safer: please let me know.

## Ansible

I created a ansible role to setup fdcs which is available here: https://github.com/mmunz/ansible-role-fdcs

## Known limitations

- in the worst case it can take 10 minutes from setting the shell for a user to when he can log in. 5 minutes for the
  froxlor tasks cronjob + 5 minutes for the fdcs cronjob.
- automatically created user homes are not removed on user deletion.

