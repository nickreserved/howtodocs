# Guide: Install vsftpd server with virtual users


## Problems

The FTP protocol is one of the oldest protocols on the internet.

Some serious disadvantages are below:


### It is not encrypted

This means that under no circumstances is it recommended to use FTP to transfer classified files.

But even worse, even user authentication is not encrypted so if there is a password, it is considered leaked.


### Works on many ports

While the default port for transferring FTP commands is 21, at least one more port is opened for data transfer. So the FTP protocol requires many ports open.

This creates a problem with some routers.

It also adds complexity for the FTP client to have to choose whether to operate in active or passive mode, as it is called.


## Advantages

For the above reasons, all major web browsers have discontinued support for the protocol. WEB is now used instead of FTP.

But there is one advantage of FTP over the WEB. FTP clients can manage the FTP filesystem hierarchy and download or upload entire folders with their files. This is not possible with any commonly accepted protocol on the WEB, which is also supported by software.

Thus, the FTP protocol is supported by almost all file managers (including Windows File Explorer) and FTP client programs.


## What we will do - Introduction

As mentioned above, it is not at all a good choice to transfer confidential files with FTP. It also makes no sense to have a password on the virtual user.

We will make filesystem hierarchies available online for public viewing.

To have independent filesystem hierarchies (e.g. Music, Videos, Books) under the same vsftpd server, we will add multiple virtual users to the vsftpd server, without a password or even with a password (but keeping in mind that the password is very easy to leak).

So, we will create a user `engineer` who will have access to the filesystem hierarchy, under the root folder `/media/nas/Engineer/` and a user `upload` who will have access to the filesystem hierarchy, under the root folder `/media/nas/upload/`.


## Installation of required packages

We ensure the installation of the required packages. Installation is for Debian or Debian based Linux.
```Shell
apt install vsftpd libpam-pwdfile
```


## Create a user for the vsftpd server

We create the user with which the vsftpd server will run:
```Shell
useradd -N -s /bin/false -d /home/vsftpd vsftpd
```


## Edit configuration files

The `/etc/pam.d/vsftpd` file defines how to authenticate virtual users of the vsftpd server:
```INI
auth required pam_pwdfile.so pwdfile /etc/vsftpd/ftpd.passwd
account required pam_permit.so
```
The `/etc/vsftpd.conf` file defines the main settings of the vsftpd server:
```INI
listen=YES
#listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
xferlog_enable=YES
# User vsftpd
nopriv_user=vsftpd
chroot_local_user=YES
# Configuration file /etc/pam.d/vsftpd
pam_service_name=vsftpd
utf8_filesystem=YES
hide_ids=YES
user_config_dir=/etc/vsftpd_user_conf
guest_enable=YES
virtual_use_local_privs=YES
guest_username=vsftpd
```
The `/etc/vsftpd/ftpd.passwd` file stores virtual users and their passwords.

If there is no code, then it is not required during authentication. But if we want the user to have a password then it is created as follows: (let's assume the password is `pavlos`):
```Shell
openssl passwd pavlos
```
Below is the file `/etc/vsftpd/ftpd.passwd` with the virtual user `engineer` without password and the user `upload` with password `pavlos`:
```INI
engineer:
upload:X7nyBRuyuJVyg
```
The `/etc/vsftpd_user_conf/engineer` file that defines the settings of the virtual user `engineer` is:
```INI
local_root=/media/nas/Μηχανικός
hide_file={/personal files,/a folder/a file}
```
While the file `/etc/vsftpd_user_conf/upload` which defines the settings of the virtual user `upload` is:
```INI
local_root=/media/nas/upload
download_enable=NO
write_enable=YES
allow_writeable_chroot=YES
```
