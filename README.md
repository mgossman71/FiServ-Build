# Linux File Server Setup (RAID0 → LVM → XFS @ `/mnt/SharedVol` + SMB/NFS + Webmin)

> This guide builds a high-throughput data volume on Ubuntu using **mdadm RAID0** across two passthrough disks, layers **LVM** on top, formats as **XFS**, mounts at **`/mnt/SharedVol`**, and shares it via **Samba (SMB)** and **NFS**. It also installs **Webmin** for UI visibility (we’re not using it to configure).

> ⚠️ **Destructive**: Steps will wipe the two selected disks. Confirm correct devices before proceeding.

---

## 0) Identify the passthrough disks (⚠️ will be wiped)

```bash
lsblk -d -o NAME,SIZE,MODEL
ls -l /dev/disk/by-id/

# Set these variables to your two disks (use /dev/disk/by-id/* for stability)
DISK1=/dev/disk/by-id/<first-disk-id>
DISK2=/dev/disk/by-id/<second-disk-id>
```

---

## 1) Install packages — Webmin (repo + install + enable)

```bash
# Clean any stale Webmin bits (safe if absent)
sudo rm -f /etc/apt/sources.list.d/webmin.list /usr/share/keyrings/webmin.gpg \
            /etc/apt/trusted.gpg.d/webmin* /etc/apt/keyrings/webmin*.gpg

# Add Webmin repo via its official helper (adds signed-by + source list)
wget https://raw.githubusercontent.com/webmin/webmin/master/webmin-setup-repo.sh
sudo sh webmin-setup-repo.sh

# Base packages (includes XFS tools, ACLs, Avahi, SMB client, and gdisk)
sudo apt update
sudo apt install -y mdadm lvm2 samba samba-vfs-modules nfs-kernel-server \
                    xfsprogs acl avahi-daemon smbclient gdisk

# Install Webmin and enable it (UI at https://<server-ip>:10000/)
sudo apt install -y webmin
sudo systemctl enable --now webmin
```

If using UFW later:
```bash
sudo ufw allow 10000/tcp
```

---

## 2) (Only if needed) Tear down any old stack before wiping

```bash
sudo systemctl stop smbd nmbd nfs-server 2>/dev/null || true
sudo umount -R /mnt/SharedVol /mnt/data 2>/dev/null || true

cat /proc/mdstat
sudo mdadm --stop /dev/md0 2>/dev/null || true
sudo mdadm --stop /dev/md127 2>/dev/null || true
sudo mdadm --remove /dev/md0 /dev/md127 2>/dev/null || true

sudo lvchange -an /dev/datavg/datalv 2>/dev/null || true
sudo vgchange -an datavg 2>/dev/null || true
sudo vgremove -y datavg 2>/dev/null || true
sudo pvremove -y /dev/md0 /dev/md127 2>/dev/null || true

sudo mdadm --zero-superblock "$DISK1" "$DISK2" 2>/dev/null || true
```

---

## 3) Wipe signatures (DESTRUCTIVE)

```bash
sudo wipefs -a -f "$DISK1"
sudo wipefs -a -f "$DISK2"
sudo sgdisk --zap-all "$DISK1"
sudo sgdisk --zap-all "$DISK2"
sudo dd if=/dev/zero of="$DISK1" bs=1M count=10 status=none
sudo dd if=/dev/zero of="$DISK2" bs=1M count=10 status=none
sudo blockdev --rereadpt "$DISK1" || true
sudo blockdev --rereadpt "$DISK2" || true
```

---

## 4) Create RAID0 as `/dev/md0` and persist it

```bash
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 "$DISK1" "$DISK2" --chunk=512 --force
cat /proc/mdstat
sudo bash -c 'mdadm --detail --scan >> /etc/mdadm/mdadm.conf'
sudo update-initramfs -u
```

---

## 5) LVM: PV → VG=`datavg` → LV=`datalv` (100%FREE)

```bash
sudo pvcreate /dev/md0
sudo vgcreate datavg /dev/md0
sudo lvcreate -l 100%FREE -n datalv datavg
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS
```

---

## 6) Make **XFS**, mount at **`/mnt/SharedVol`**, add to **fstab**

> Optional tuning for RAID0 (2 disks, 512K chunk): `-d su=512k,sw=2`

```bash
sudo mkfs.xfs -f -m crc=1 -d su=512k,sw=2 /dev/datavg/datalv

sudo mkdir -p /mnt/SharedVol
UUID=$(sudo blkid -s UUID -o value /dev/datavg/datalv)
echo "UUID=${UUID} /mnt/SharedVol xfs defaults,noatime 0 0" | sudo tee -a /etc/fstab
sudo mount -a
findmnt /mnt/SharedVol
```

---

## 7) Cooperative permissions + default ACLs

```bash
sudo groupadd -f fileshare
sudo chgrp -R fileshare /mnt/SharedVol
sudo chmod 2775 /mnt/SharedVol

# POSIX ACLs for consistent group write (and inheritance)
sudo setfacl -R -m g:fileshare:rwx /mnt/SharedVol
sudo setfacl -R -m d:g:fileshare:rwx /mnt/SharedVol

# Add your Linux user(s) to the group (example: gozz)
sudo usermod -aG fileshare gozz
# Re-login (or: newgrp fileshare)
```

Quick check:
```bash
getfacl /mnt/SharedVol
```

---

## 8) Samba configuration (share name **SharedVol**, path **/mnt/SharedVol**)

```bash
# Backup current config
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak.$(date +%F-%H%M%S)

# Disable printer shares (optional; avoids mixed vfs_fruit warning)
sudo sed -i '/^\[printers\]/,/^\[/ s/^/#/' /etc/samba/smb.conf
sudo sed -i '/^\[print\$\]/,/^\[/ s/^/#/' /etc/samba/smb.conf

# Ensure helpful globals
sudo tee -a /etc/samba/smb.conf >/dev/null <<'EOF'

[global]
   vfs objects = acl_xattr fruit streams_xattr
   ea support = yes
   store dos attributes = yes
   map acl inherit = yes
   inherit acls = yes
   fruit:metadata = stream
   fruit:resource = stream
EOF

# Define the share
sudo tee -a /etc/samba/smb.conf >/dev/null <<'EOF'

[SharedVol]
   path = /mnt/SharedVol
   browseable = yes
   read only = no
   force group = fileshare
   create mask = 0664
   directory mask = 02775
   veto files = /._*/.DS_Store/Thumbs.db/
   vfs objects = acl_xattr fruit streams_xattr
   ea support = yes
   store dos attributes = yes
   inherit acls = yes
   map acl inherit = yes
EOF

# Validate & start
testparm -s | sed -n '/^\[SharedVol\]/,/^\[/p'
sudo systemctl enable --now smbd
sudo systemctl reload smbd

# Create SMB credentials for your user(s)
sudo smbpasswd -a gozz

# Local test
smbclient -L localhost -U gozz
smbclient //localhost/SharedVol -U gozz -c "ls"
```

---

## 9) Windows discovery (WS-Discovery)

```bash
sudo apt install -y wsdd || true

# If the wsdd.service unit is missing, create one:
if ! systemctl list-unit-files | grep -q '^wsdd\.service'; then
  which wsdd || command -v wsdd
  sudo tee /etc/systemd/system/wsdd.service >/dev/null <<'EOF'
[Unit]
Description=Web Services Dynamic Discovery host daemon (wsdd)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/wsdd --shortlog --ipv4only --workgroup WORKGROUP
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
  sudo systemctl daemon-reload
fi

sudo systemctl enable --now wsdd
```

You should see it on **TCP 5357** and **UDP 3702**:
```bash
ss -lntup | grep -E '(:5357|:3702)' || true
```

---

## 10) macOS discovery (Bonjour/mDNS)

```bash
sudo systemctl enable --now avahi-daemon

sudo tee /etc/avahi/services/samba.service >/dev/null <<'EOF'
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
  <name replace-wildcards="yes">%h</name>
  <service>
    <type>_smb._tcp</type>
    <port>445</port>
  </service>
  <service>
    <type>_device-info._tcp</type>
    <port>0</port>
    <txt-record>model=RackMac</txt-record>
  </service>
</service-group>
EOF

sudo systemctl restart avahi-daemon
```

---

## 11) NFS export (path **/mnt/SharedVol**, adjust LAN)

```bash
echo '/mnt/SharedVol 10.0.0.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl enable --now nfs-server
sudo exportfs -v
```

---

## 12) (If using UFW) allow services

```bash
sudo ufw allow 10000/tcp           # Webmin
sudo ufw allow Samba
sudo ufw allow 5353/udp            # mDNS (Avahi)
sudo ufw allow 5357/tcp 3702/udp   # WSD (wsdd)
sudo ufw allow from 10.0.0.0/24 to any port nfs
sudo ufw status
```

---

## 13) Final verification

```bash
# Storage
cat /proc/mdstat
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS
findmnt /mnt/SharedVol
getfacl /mnt/SharedVol

# Samba
testparm -s | sed -n '/^\[SharedVol\]/,/^\[/p'
smbclient -L localhost -U gozz
smbclient //localhost/SharedVol -U gozz -c "ls"

# Discovery
ss -lntup | grep -E '(:5357|:3702)' || true
avahi-browse -at | grep -i smb || true

# Webmin
ss -lntp | grep 10000 || sudo netstat -lntp | grep 10000
curl -k https://127.0.0.1:10000/ | head -n 5

# NFS
sudo exportfs -v
```

---

## Client quick links

- **Windows:** `\\<server>\SharedVol`
- **macOS:** Finder → Go → Connect to Server → `smb://<server>/SharedVol` (or `.local`)
- **Linux (SMB):** `sudo mount -t cifs //server/SharedVol /mnt -o username=<user>`
- **Linux (NFS):** `sudo mount -t nfs server:/mnt/SharedVol /mnt`

---

### Notes
- RAID0 has **no redundancy**; ensure you have backups.
- XFS defaults are great for large files/VM images; stripe hints provided.
- ACLs + `acl_xattr/fruit/streams_xattr` make SMB behave nicely for Windows/macOS while keeping POSIX semantics server-side.
