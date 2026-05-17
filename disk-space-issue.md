# Disk Space Expansion Updates at Ubuntu

When we expand the disk space from VM, the Ubuntu is not recognized newly allocated disk space in sda. We need to manually configure it.

First, let's prove that Ubuntu actually sees the 50GB disk. Run this command:

```bash
lsblk
```

You should see sda listed at the top with 50G, and underneath it, sda5 still stuck at around 29G. This confirms the space is there, just unallocated.


Ubuntu usually comes with a tool called growpart pre-installed, but let's make absolutely sure you have it:

```bash
sudo apt update
sudo apt install cloud-guest-utils -y
```

Now we tell the partition table to stretch /dev/sda5 into the empty 20GB of space.

(Note: There is a space between sda and 5 in this command. This is intentional!)

```bash
sudo growpart /dev/sda 5
```

If it is not working

```bash
sudo growpart /dev/sda 2
sudo growpart /dev/sda 5
```

You have expanded the partition (the container), but now you have to expand the actual filesystem (the formatting) so the OS can write files to it.

Assuming you are using the default ext4 filesystem, run:

```bash
sudo resize2fs /dev/sda5
```

Once resize2fs finishes (it should be nearly instant), check your disk space one more time:

```bash
df -h /
```