
### Gist: Cloning and Restoring Ubuntu System with LUKS Encryption

#### Boot your computer with a Live Ubuntu USB - All the following steps will not work if you are within your running installation. The "source", which is typically your internal drive, should be mounted from an outside installation, therefore a Live Ubuntu USB is definitely the easiest way.

#### Creating the Backup


1. **Open Source LUKS Container**
   - `sudo cryptsetup open /dev/sdX source_crypt`
   - Replace `/dev/sdX` with your source disk's device identifier.

2. **Mount Source Filesystem**
   - `sudo mkdir -p /mnt/source`
   - `sudo mount /dev/mapper/source_crypt /mnt/source`

3. **Initialize Destination LUKS Container**
   - `sudo cryptsetup luksFormat /dev/sdY`
   - `sudo cryptsetup open /dev/sdY destination_crypt`
   - Replace `/dev/sdY` with your destination disk's device identifier.

4. **Mount Destination Filesystem**
   - `sudo mkdir -p /mnt/destination`
   - `sudo mount /dev/mapper/destination_crypt /mnt/destination`

5. **Rsync Data**
   - `sudo rsync -aHAXxv --numeric-ids --info=progress2 /mnt/source/ /mnt/destination/`

6. **Unmount and Close Containers**
   - `sudo umount /mnt/source /mnt/destination`
   - `sudo cryptsetup close source_crypt destination_crypt`

#### Restoring the Backup to a New Drive

1. **Prepare New Disk with LUKS**
   - `sudo cryptsetup luksFormat /dev/sdZ`
   - `sudo cryptsetup open /dev/sdZ newdisk_crypt`
   - Replace `/dev/sdZ` with your new disk's device identifier.

2. **Mount New Disk Filesystem**
   - `sudo mkdir -p /mnt/newdisk`
   - `sudo mount /dev/mapper/newdisk_crypt /mnt/newdisk`

3. **Restore Data with Rsync**
   - Repeat the rsync command as before to clone data from the backup to the new disk.

4. **Chroot into New System**
   - Bind necessary directories:
     ```bash
     for dir in /dev /dev/pts /proc /sys /run; do sudo mount --bind $dir /mnt/newdisk$dir; done
     ```
   - `sudo chroot /mnt/newdisk`

5. **Install and Update GRUB**
   - `grub-install /dev/sdZ`
   - `update-grub`
   - Replace `/dev/sdZ` with your new disk's device identifier.

6. **Update `/etc/fstab`**
   - Inside chroot, use `blkid` to find the UUIDs of the new disk's partitions.
   - Open `/etc/fstab` with a text editor like `nano` or `vi`.
   - Replace the old UUIDs with the new ones for each partition (root, home, swap, etc.).
   - Ensure that the file paths and mount options are correct.
   - Save and exit the editor.

7. **Exit Chroot and Unmount**
   - `exit`
   - `for dir in /dev/pts /dev /proc /sys /run; do sudo umount /mnt/newdisk$dir; done`
   - `sudo umount /mnt/newdisk`

8. **Close New Disk's LUKS Container**
   - `sudo cryptsetup close newdisk_crypt`

### Additional Notes:

- **Testing**: Boot from the new disk to ensure the system runs correctly.
- **Regular Backups**: Keep your backup updated regularly.
- **Secure Storage**: Store the backup disk in a secure location.

This gist provides a detailed guide on cloning an Ubuntu system with LUKS encryption to a backup drive and restoring it to a new drive, including making the new drive bootable. Each step is crucial, so proceed with caution and ensure you understand each command before executing it.
