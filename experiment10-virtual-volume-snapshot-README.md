# Experiment 10: Virtual Volume and Snapshot Using a GitHub Codespace

## Aim
To create a virtual block volume, attach it, format it with the ext4 file system, store a file, create a snapshot, delete the original file, and recover it by restoring the snapshot—without requiring an AWS account.

## Task 1: Create a New Volume

```bash
cd ~
truncate -s 64M my-volume.img
ls -lh my-volume.img
```

This creates a 64 MB disk-image file named `my-volume.img`.

## Task 2: Attach the Volume

```bash
LOOP=$(sudo losetup --find --show ~/my-volume.img)
echo "$LOOP"
lsblk
```

The variable `LOOP` stores the loop-device name, such as `/dev/loop5`. It acts as the local equivalent of a cloud block device such as `/dev/sdf`.

## Task 3: View the Current Storage

```bash
df -h
```

The new device does not appear in the mounted-filesystem list yet because it has not been formatted and mounted.

## Task 4: Configure the File System

```bash
sudo mkfs -t ext4 "$LOOP"
sudo mkdir /mnt/data-store
sudo mount "$LOOP" /mnt/data-store
df -h
sudo sh -c "echo some text > /mnt/data-store/file.txt"
cat /mnt/data-store/file.txt
```

The commands format the volume as ext4, mount it at `/mnt/data-store`, and create `file.txt`.

Expected file contents:

```text
some text
```

**Note about `/etc/fstab`:** The original manual may ask you to add an `/etc/fstab` entry so the volume mounts after reboot. This step is not required for this GitHub Codespace experiment; it can be discussed as a viva question.

## Task 5: Create a Snapshot

Unmount the volume before copying the image so the snapshot is consistent.

```bash
sudo umount /mnt/data-store
cp --sparse=always ~/my-volume.img ~/my-snapshot.img
ls -lh ~/my-snapshot.img
```

Mount the original volume again, delete the file, and verify that it is gone:

```bash
sudo mount "$LOOP" /mnt/data-store
sudo rm /mnt/data-store/file.txt
ls /mnt/data-store/
```

At this point, `file.txt` should no longer be listed in the original volume.

## Task 6: Restore from the Snapshot

Create a new image from the snapshot, attach it, and mount it separately.

```bash
cp --sparse=always ~/my-snapshot.img ~/restored-volume.img
LOOP2=$(sudo losetup --find --show ~/restored-volume.img)
sudo mkdir /mnt/data-store2
sudo mount "$LOOP2" /mnt/data-store2
ls /mnt/data-store2/
cat /mnt/data-store2/file.txt
```

Expected output:

```text
file.txt  lost+found
```

Contents of `file.txt`:

```text
some text
```

The file is present in the restored volume even though it was deleted from the original volume. This demonstrates recovery from a snapshot.

## Fallback: If Loop Mounting Is Not Permitted

If `losetup` or `mount` reports a permission error or a “no loop device” error in the Codespace, use `debugfs` to work directly with the ext4 image. This fallback does not require mounting.

```bash
truncate -s 64M my-volume.img
mkfs.ext4 -F my-volume.img

echo "some text" > file.txt
debugfs -w -R "write file.txt file.txt" my-volume.img
debugfs -R "ls" my-volume.img

cp my-volume.img my-snapshot.img
debugfs -w -R "rm file.txt" my-volume.img
debugfs -R "ls" my-volume.img
# file.txt should be gone from the original image

cp my-snapshot.img restored-volume.img
debugfs -R "cat file.txt" restored-volume.img
# Expected output: some text
```

## Clean-up

Run these commands only after completing the experiment and saving any required evidence.

```bash
sudo umount /mnt/data-store /mnt/data-store2
sudo losetup -d "$LOOP" "$LOOP2"
sudo rmdir /mnt/data-store /mnt/data-store2
rm ~/my-volume.img ~/my-snapshot.img ~/restored-volume.img
```

If you used the `debugfs` fallback, clean up its files separately if they are no longer needed:

```bash
rm my-volume.img my-snapshot.img restored-volume.img file.txt
```

## Full Command Reference

For quick reference, the main loop-device workflow is:

```bash
truncate -s 64M ~/my-volume.img
LOOP=$(sudo losetup --find --show ~/my-volume.img)
sudo mkfs -t ext4 "$LOOP"
sudo mkdir /mnt/data-store && sudo mount "$LOOP" /mnt/data-store
sudo sh -c "echo some text > /mnt/data-store/file.txt"
sudo umount /mnt/data-store
cp --sparse=always ~/my-volume.img ~/my-snapshot.img
sudo mount "$LOOP" /mnt/data-store && sudo rm /mnt/data-store/file.txt
cp --sparse=always ~/my-snapshot.img ~/restored-volume.img
LOOP2=$(sudo losetup --find --show ~/restored-volume.img)
sudo mkdir /mnt/data-store2 && sudo mount "$LOOP2" /mnt/data-store2
ls /mnt/data-store2/
cat /mnt/data-store2/file.txt
```

## Result

A virtual block volume was created, attached, formatted, mounted, and used to store a file inside a GitHub Codespace. A snapshot was taken, the file was deleted from the original volume, and the data was recovered by restoring a new volume from the snapshot.

The experiment reproduces the core outcome of an Amazon EBS volume snapshot workflow without requiring an AWS account.
