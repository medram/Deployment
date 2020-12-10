# Django Deployment to Ubuntu

## Server Security & Access
### Creating a new User and asign a `sudo` group to it:
```
$ useradd -m nabil
$ usermod -aG sudo nabil
$ passwd nabil
```

### SSH Configuration
Creating a SSH key
```
$ ssh-keygen -t rsa -b 4096 -F customfilename
```

Move the public key to the server:
First method:
```
$ scp ~/.ssh/public_key.pub user@server-IP:~/.ssh/public_key.pub
```
then from server side, the public key needs to be appended to `authorized_keys` file:
```
$ cd ~/.ssh
$ cat public_key.pub >> authorized_keys
$ rm public_key.pub
```
changing permissions for `.ssh` folder (required step):
```
$ sudo chmod 700 ~/.ssh
$ sudo chmod 600 ~/.ssh/*
```

Second method which is easier:
```
$ ssh-copy-id -i ~/.ssh/public_key.pub user@server-IP
```

Change ssh configuration:
```
$ sudo nano /etc/ssh/sshd_config
```

then change these parameters:
```
Port 2222                  # whatever port you want (shouldn't be reserved)

AllowUsers nabil           # change `nabil` with whatever user created
AllowGroups nabil          # change `nabil` with whatever user created

PermitRootLogin no         # Permit the root account from using ssh
PasswordAuthentication no  # Don't use Passwords to authenticate (just public keys)
```
Restart SSH server:
```
$ sudo systemctl restart sshd
```
### Firewall Configuration:
installing `ufw` if doesn't exist
```
$ sudo apt install ufw
```
setting up permissions
```
$ sudo ufw default allow outgoing
$ sudo ufw default deny incoming
```
to allow ssh ports run this:
```
$ sudo ufw allow ssh
```
but for a custom ssh port (e.g: 2222) run this:
```
$ sudo ufw allow 2222
```
allowing a test port 5000:
```
$ sudo ufw allow 5000
```

to enabling & check ufw:
```
$ sudo ufw enable
$ sudo ufw status
```
































