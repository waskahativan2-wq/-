# Persistent USB binding and 16K cluster formatting

## 1) Persistent USB passthrough to a VM (udev + systemd + libvirt)

### Find USB IDs
With the USB drive connected:

```bash
lsusb
```

Find the entry `ID 1234:5678` (`1234` = `idVendor`, `5678` = `idProduct`).

### Create a udev rule
Create `/etc/udev/rules.d/99-retro-usb.rules`:

```udev
SUBSYSTEM=="usb", ATTR{idVendor}=="1234", ATTR{idProduct}=="5678", SYMLINK+="retro_usb_drive", TAG+="systemd"
```

Reload udev:

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

### Create VM hotplug script
Create `/usr/local/sbin/vm-usb-bind.sh`:

```bash
#!/bin/bash
ACTION=$1
VM_NAME="your-vm-name"

if [ "${ACTION}" == "add" ]; then
  CMD="attach-device"
else
  CMD="detach-device"
fi

virsh ${CMD} ${VM_NAME} /dev/stdin <<EOF
<hostdev mode='subsystem' type='usb' managed='yes'>
  <source>
    <vendor id='0x1234'/>
    <product id='0x5678'/>
  </source>
</hostdev>
EOF
```

Make it executable:

```bash
sudo chmod +x /usr/local/sbin/vm-usb-bind.sh
```

### Add systemd service
Create `/etc/systemd/system/vm-usb-bind@.service`:

```ini
[Unit]
Description=Auto-bind USB to VM
BindsTo=dev-retro_usb_drive.device
After=dev-retro_usb_drive.device

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/vm-usb-bind.sh add
ExecStop=/usr/local/sbin/vm-usb-bind.sh remove

[Install]
WantedBy=multi-user.target
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

## 2) Format a 128GB drive with 16K clusters

Check your target partition first:

```bash
lsblk
```

Then replace `/dev/sdX1` below.

### exFAT (recommended for broad compatibility)
```bash
sudo mkfs.exfat -c 16K /dev/sdX1
```

### NTFS
```bash
sudo mkfs.ntfs -Q -c 16384 /dev/sdX1
```

### FAT32
```bash
sudo mkfs.vfat -F 32 -s 32 /dev/sdX1
```

### ext4 note
On most stock Linux systems, ext4 generally uses up to 4K blocks by default. True 16K-equivalent allocation behavior requires non-default features/support and is typically not the practical choice for this requirement.

## 3) Handheld OS usage note

For AmberELEC, OnionOS, and GarlicOS use cases, exFAT with 16K clusters is usually the safest first option for compatibility.
