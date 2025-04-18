# Server Setup
I have an old Microsoft Surface Book 2 that I don't use anymore and that isn't really good for much as far as computers go, so I wiped the windows operating system and replaced it with an Ubuntu Server.

## Replacing Windows with Ubuntu Server (on Surface Book 2)

This process is specific to replacing windows on the Microsoft Surface Book 2, but is similar for all other hardware I believe.

1. Create a bootable USB drive.
    1. Download the appropriate `.iso` file from [ubuntu.com](https://ubuntu.com).
    2. Insert a USB drive into your computer.
    3. Create the bootable usb (on linux) by doing the following:
        1. Find the mount point of your USB by searching the results of
        ```sh
        sudo fdisk -l
        # e.g. if you see /dev/sda1 then your USB is mounted at /dev/sda (without the number)
        ```
        2. Run
        ```sh
        sudo dd if=/your/iso/file/path of=/dev/your-usb bs=4M status=progress conv=fsync
        ```
        3. (For windows) use Rufus
2. Enter the BIOS/UEFI for the Surface Book 2 by holding **power** and **volume up** while booting. Release when the windows logo shows up so you don't just turn it off again. In UEFI, look for some kind of boot settings and make sure USB boot is highest priority.
3. Ensure settings are saved and restart computer. It will take you through setting up Ubuntu Server. Make sure you select that you want to wipe the hard drive and replace it with Ubuntu Server.
    * When I set this up, I left basically all settings as the default and I didn't have an internet connection.

## Connecting to Wifi
I couldn't get a direct ethernet connection at first, so I used `netplan` to connect to wifi. To do this: open or create a netplan config file:

```bash
sudo nvim /etc/netplan/01-wifi.yaml
```

It should look like this, replacing obvious things with your actual wifi info:
```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlp1s0:
      dhcp4: true
      optional: true
      access-points:
        "YourSSID":
          password: "YourPassword"
```

### Aside about permissions
To make it so that only the file owner can access this file, and that the file owner is root, run these:
```bash
chmod 600 /etc/netplan/01-wifi.yaml
chown root:root /etc/netplan/01-wifi.yaml
```

## Setting up `ssh`
Then I set up `openssh` so I could at least connect to it on the local network. Pretty simple stuff:
1. Run:
```sh
sudo apt install openssh
```
2. Make sure its running:
```sh
sudo systemctl enable ssh
```
3. Generate and public key from client devices.
4. (Optional) edit `/etc/ssh/sshd_config` to enforce password and ssh authentication.

## Tailscale
For the moment, Google Fiber is trying to fix an actual bug with our wifi, so I can't access the router to set up port forwarding. In temporary lieu of this, I'm using `tailscale` to access my server from wherever I want. It just can't be public yet. Here's the rundown:

Run this on your server and any client you want to be a part of your private network:

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Visit the link it tells you to and sign up for a free account. It's not ideal, but it's good enough until we can get port forwarding set up. More commands:

```sh
sudo tailscale up   # start tailscale service on your device & authenticate
tailscale status    # see current connection status and list of connected devices
sudo tailscale down # disconnect tailscale service
```

# Setting up a website (web server)

## Nginx
Nginx seems to be generally agreed upon as the better option for me since I don't use anything that requires apache specifically, and my setup will be pretty straightforward I think. To set up:

```sh
sudo apt update
sudo apt install nginx -y # -y is to also install dependencies


# start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx # enable means it runs at boot

# check status
systemctl status nginx
```

## Firewall edits
You might need to configure your firewall also:
```sh
sudo ufw allow 'Nginx Full'
```

## Nginx config
Nginx config can be set by:
```sh
nvim /etc/nginx/sites-available/default
```

I set my root directory to be `/var/www/public`.

## Git repo
### Server-side setup
To update the web repo, I'm using a **bare git repository**, which is a repository that doesn't have a working tree. I set it up like so:
```sh
mkdir -p /var/repos/moosebox.git
cd /var/repos/moosebox.git
git init --bare
```

This puts a bunch of files in there by default, but what you really want is the `hooks` directory. Create a new hook and it will run every time you push something to it. To have it update my web server, I created the following web hook:
```sh
GIT_WORK_TREE=/var/www git checkout -f
```

* A bare directory doesn't have a working tree, so by setting `GIT_WORK_TREE`, I'm directing Git to update files in a different directory.
* Then, `git checkout -f` forces Git to check out the current branch's contents into the working tree we just defined.

### Developer/client-side setup
Once you have the production git repo set up, you want to create a repo on your local machine and add your serers bare git repo as a remote repo:
```
# once you've created the repo
git remote add production ssh://username@moosebox.net/var/repos/moosebox.git
```

* Using `production` here just tells git that when you refer to `production` you're referring to the repository you just specified. Most of the time up to this point, you've just used `origin`. You're just creating an **alias** for the server.
