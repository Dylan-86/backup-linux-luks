
### **Cloning and Restoring Ubuntu System with LUKS Encryption**

#### **Prerequisite:**

-   Boot your computer with a Live Ubuntu USB.
-   Ensure you're not inside the running installation. Use the Live USB environment to access your internal drive, referred to as the "source."

----------

### **Creating the Backup**

#### 1. **Open Source LUKS Container**

Use the following steps to open your LUKS-encrypted container:

List all devices to identify your source disk.

 1. `df`  take note of the right /dev/sdX of the hard disk to clone
 3. `sudo cryptsetup open /dev/sdX source_crypt` 

Replace `/dev/sdX` with your source disk's identifier.

#### 2. **Mount Source Filesystem**

Mount the source disk's filesystem to `/mnt/source`:

`sudo mkdir -p /mnt/source`
`sudo mount /dev/mapper/source_crypt /mnt/source` 

#### 3. **Prepare Destination Drive**
- use `lsblk` to see all the connected devices and take note of the identifier of your external drive/destination disk
-   If your external (backup) drive is **new**, go to step 4.
-   If you're updating an existing **backup**, go to step 5.

#### 4. **NEW EXTERNAL DRIVE: Initialize Destination LUKS Container**

Format the new drive with LUKS encryption and open it:

`sudo cryptsetup luksFormat /dev/sdY`  # Replace `/dev/sdY` with your destination disk's identifier.
`sudo cryptsetup open /dev/sdY destination_crypt`

After this, skip to step 6.

#### 5. **EXISTING EXTERNAL DRIVE: Open Existing LUKS Container**

If your external drive is already formatted with LUKS, follow these commands to mount it:

`sudo vgchange -ay`  # Activate LVM - needed to make the system recognize the LUKS filesystem.
`sudo cryptsetup open /dev/sdc1 external_crypt`  # Replace `/dev/sdc1` with the correct partition.
`sudo lvscan`  # Find the LVM partitions (typically `vgubuntu/root`).
`sudo mount /dev/vgubuntu/root /mnt/source`

#### 6. **Mount Destination Filesystem**


`sudo mkdir -p /mnt/destination` 
`sudo mount /dev/mapper/destination_crypt /mnt/destination`

#### 7. **Rsync Data**

Use `rsync` to clone your source system to the destination. 
You may remove the `--delete` option if you do not want to delete files on the destination that no longer exist on the source.
Note that this step may take several hours.

`sudo rsync -aHAXxv --numeric-ids --delete --info=progress2 /mnt/source/ /mnt/destination/` 

#### 8. **Unmount and Close Containers**

When the `rsync` process is complete, unmount and close both containers:

`sudo umount /mnt/source /mnt/destination`
`sudo cryptsetup close source_crypt destination_crypt` 

----------

### **Restoring the Backup to a New Drive**
These steps will let you restore the backup from the external drive to an internal hard disk / SSD.

#### 1. **Prepare New Disk with LUKS**

Format the new drive (internal hard drive for example) with LUKS encryption and open it:
`df` # List all the internal drives, take note of the ID of the one you need to use. The backup will be restored there.
This step formats the disk.

`sudo cryptsetup luksFormat /dev/sdZ`  # Replace `/dev/sdZ` with your new disk's identifier.
`sudo cryptsetup open /dev/sdZ newdisk_crypt`

#### 2. **Mount New Disk Filesystem**

`sudo mkdir -p /mnt/newdisk`
`sudo mount /dev/mapper/newdisk_crypt /mnt/newdisk` 

#### 3. **Restore Data with Rsync**

Use the same `rsync` command as in the backup process to restore data to the new disk:

`sudo rsync -aHAXxv --numeric-ids --info=progress2 /mnt/destination/ /mnt/newdisk/` 

There is no need to use --delete since the drive will be empty.

#### 4. **Chroot into New System**

To make the new system bootable, you'll need to `chroot` into it and reinstall GRUB:


`for dir in /dev /dev/pts /proc /sys /run; do sudo mount --bind $dir /mnt/newdisk$dir; done`

`sudo chroot /mnt/newdisk` 

#### 5. **Install and Update GRUB**



`grub-install /dev/sdZ`  # Replace `/dev/sdZ` with your new disk's identifier.
`update-grub`

#### 6. **Update `/etc/fstab`**

Within the `chroot` environment, update `/etc/fstab`:

1.  Use `blkid` to list the UUID and find the UUIDs of the new disk's partitions.
This command will output something like:

`/dev/sda1: UUID="d1f2fcd6-02ef-4ad6-9f08-bf282f2547f2" TYPE="ext4" PARTUUID="3b1f6f3e-01"`
`/dev/sda2: UUID="7156-9B0D" TYPE="vfat" PARTUUID="3b1f6f3e-02"`
`/dev/sda3: UUID="68bc7263-b28c-4735-b62e-295c3dd9eb9e" TYPE="swap" PARTUUID="3b1f6f3e-03"`


-   **`/dev/sda1`**: This could be your root (`/`) partition, with the `UUID="d1f2fcd6-02ef-4ad6-9f08-bf282f2547f2"`.
-   **`/dev/sda2`**: This might be your EFI partition (if using UEFI boot), with `UUID="7156-9B0D"`.
-   **`/dev/sda3`**: This could be your swap partition with its unique UUID.

Take note of these UUIDs for the partitions on your new disk.

2.  Open `/etc/fstab` with an editor (e.g., `nano` or `vi`):


`nano /etc/fstab` 
It will look more or less like this

    # <file system> <mount point> <type> <options> <dump> <pass>
    UUID=old-root-uuid  /      ext4  errors=remount-ro  0       1
    UUID=old-efi-uuid   /boot/efi  vfat  defaults         0       1
    UUID=old-swap-uuid  none   swap  sw                 0       0

3.  Replace the old UUIDs with the new ones for each partition (root, home, swap, etc.).
4.  Save and exit the editor. (ctrl + o to save, crtl +x to exit).

#### 7. **Exit Chroot and Unmount**

`exit`  # Exit the chroot environment.

`for dir in /dev/pts /dev /proc /sys /run; do sudo umount /mnt/newdisk$dir; done`

`sudo umount /mnt/newdisk` 

#### 8. **Close New Disk's LUKS Container**

`sudo cryptsetup close newdisk_crypt`



**Now you should be able to reboot normally into the new disk.**
