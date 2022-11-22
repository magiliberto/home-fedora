# fedora-install
Source:
[Video](https://www.youtube.com/watch?v=nDDVMaOKXyU)
[Docker](https://docs.docker.com/engine/install/fedora/)
[R](https://cran.r-project.org/bin/linux/fedora/)

## Prepare installation media
```bash
release=37
arch=x86_64
ver=1.7
iso=Fedora-Workstation-Live-$arch-$release-$ver.iso

cd $HOME/Downloads
wget "https://download.fedoraproject.org/pub/fedora/linux/releases/$release/Workstation/$arch/iso/$iso" -O $iso

curl -O https://getfedora.org/static/checksums/37/iso/Fedora-Workstation-$release-$ver-$arch-CHECKSUM

curl -O https://getfedora.org/static/fedora.gpg

gpgv --keyring ./fedora.gpg *-CHECKSUM

sha256sum -c *-CHECKSUM

sudo dd if=$iso of=/dev/sdb bs=8M status=progress oflag=direct
```

## GUI Install

### Partition Table

name | partition | type | size | mount |
--- | --- | --- | --- | --- |
EFI | partition | EFI partition | 512 mib | /boot/efi |
FEDORA | btrfs |  | 100 gib | / |
FILES | partition | ext4 | rest | /media/cloud |

### Subvolumes

name | mount | type |
--- | --- | --- |
[main] | / | mainvolume |
opt | /opt | subvolume |
tmp | /tmp | subvolume |
var | /var | subvolume |
usr-local | /usr/local | subvolume |
snapshots | /.snapshots | subvolume |

## After Install

1 - snap base installation
2 - install software
3 - set gdrive folder with InSync & allow KeePassXC SSH AGENT INTEGRATION
4 - clone home-fedora
5 - run script to generate folder sctructure and set XDG and stow

### 1 - snap base installation
```bash
sudo dnf update

chattr -R -f +C /var
lsattr -d /var

echo "GRUB_ENABLE_CRYPTODISK=y" | sudo tee -a /etc/default/grub
echo "SUSE_BTRFS_SNAPSHOT_BOOTING=\"true\"" | sudo tee -a /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg

lsblk -pf # copy ID from encrypted Partition
vi /boot/efi/EFI/fedora/grub.cfg
# add "set btrfs_relative_path="yes""
# add "cryptomount -u <clipboard>" remove "-"

sudo dnf install git snapper
sudo umount /.snapshots
sudo rm -rf /.snapshots
sudo snapper -c root create-config /
sudo btrfs subvolume list / # A new .snapshots subvol is created by snapper
sudo btrfs subvolume delete /.snapshots # Delete it as not needed
sudo mkdir /.snapshots
sudo systemctl daemon-reload
sudo mount -a
lsblk -p
sudo snapper -c root set-config ALLOW_USER=$USER SYNC_ACL=yes
sudo chown -R :$USER /.snapshots
sudo grub2-editenv - unset menu_auto_hide 

git clone https://github.com/Antynea/grub-btrfs.git
cd grub-btrfs
sudo make install
sudo vi /etc/default/grub-btrfs/config
# uncomment GRUB_BTRFS_SHOW_TOTAL_SNAPSHOTS_FOUND="true"
# uncomment GRUB_BTRFS_BOOT_DIRNAME="/boot/grub2"
# uncomment GRUB_BTRFS_MKCONFIG=/usr/sbin/grub2-mkconfig
# uncomment GRUP_BTRFS_CHECK=grub2-script-check
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo systemctl enable --now grub-btrfs.path
cd $HOME
rm -rf ./grub-btrfs

sudo snapper -c root create --description="after-base-installation"
reboot
```

commands:
```bash
btrfs subvol list -p /
lsblk -p
cat /etc/fstab
mount -va
snapper list-configs
snapper -c root list
snapper ls
```

### 2 - install software
```bash
sudo dnf -y update
sudo dnf -y install dnf-plugins-core fedora-workstation-repositories
sudo dnf install stow alacritty tmux neovim toolbox keepassxc htop ranger 
sudo dnf install conda R R-flexiblas flexiblas-* rstudio-desktop rstudio-server

# Chrome
sudo dnf config-manager --set-enabled google-chrome
sudo dnf install google-chrome-stable

# Brave
sudo rpm --import https://brave-browser-rpm-release.s3.brave.com/brave-core.asc
sudo dnf config-manager --add-repo https://brave-browser-rpm-release.s3.brave.com/x86_64/
sudo dnf install brave-browser 

# Insync
sudo rpm --import https://d2t3ff60b2tol4.cloudfront.net/repomd.xml.key
echo "[insync]
name=insync repo
baseurl=http://yum.insync.io/fedora/\$releasever/
gpgcheck=1
gpgkey=https://d2t3ff60b2tol4.cloudfront.net/repomd.xml.key
enabled=1
metadata_expire=120m" | sudo tee -a /etc/yum.repos.d/insync.repo
sudo yum install insync

# Docker
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo systemctl start docker
sudo docker run hello-world

sudo snapper -c root create --description="after-software-installation"
reboot
```

### 3 - set gdrive folder with InSync & allow KeePassXC SSH AGENT INTEGRATION
```bash
sudo chown mg:mg /media/cloud
sudo mkdir /media/cloud/google-drive
sudo ln -s /media/cloud/google-drive $HOME/gdrive
insync start
# Use insync GUI to link google-drive account to $/HOME/gdrive
# Sync $HOME/gdrive/keepassxc/keepass-mg.kdbx
# Use KeePassXC GUI to read $HOME/gdrive/keepassxc/keepass-mg.kdbx
# Allow SSH AGENT INTEGRATION

sudo snapper -c root create --description="after-gdrive-sync"
reboot
```

### 4 - clone GitHub repos
```bash
mkdir $HOME/repos
cd $HOME/repos
repos=$(curl "https://api.github.com/users/magiliberto/repos?per_page=100" | grep -o 'git@[^"]*')
for r in $repos;do
	git clone $r
done

ln -s $HOME/gdrive/notes $HOME/repos/home-fedora/notes/notes-private
notes=$(ls $HOME/repos | grep notes-)
for n in $notes;do
	ln -s $HOME/repos/$n $HOME/repos/home-fedora/notes/$n
done

cd $HOME/repos/home-fedora
stow --adopt -vSt ~ bin
stow --adopt -vSt ~ notes
```

## Folder Structure

-mg/
--tmp/
---desktop/
---downloads/ # config browsers
---screenshots/ # config screenshot tool
---pictures/ # config cheese tool

--gdrive -> /media/cloud/google-drive/ # InSync Synched and Encrypted partition
---music # config XDG
---pictures
---documents
---videos
---shared
---library
---notes
|

--home-fedora/ # GitHub remote
---bin/bin/
---notes/notes/
----.gitignore "notes-private notes-"
----notes-private -> $HOME/gdrive/notes
----notes-quant -> $HOME/repos/notes-quant # generated by notes-clone script
----notes-time-series -> $HOME/repos/notes-time-series # generated by notes-clone script
---.dotfiles/
----bash/
----tmux/
|
