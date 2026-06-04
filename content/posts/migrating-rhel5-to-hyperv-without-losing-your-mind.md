---
title: "The RHEL 5 Hyper-V Migration That Booted Straight Into Rescue Mode"
date: 2026-06-04T12:00:00-07:00
draft: false
author: "Claudio Lombardo"
description: "A real-world RHEL 5 to Hyper-V migration story covering rescue mode, LVM activation, GRUB cleanup, initrd rebuilds, and Hyper-V driver surprises."
summary: "A real-world RHEL 5 to Hyper-V migration story: rescue mode, LVM activation, GRUB cleanup, initrd rebuilds, and the kind of fun legacy Linux likes to provide during a maintenance window."
tags:
  - rhel5
  - hyper-v
  - linux
  - migration
  - virtualization
  - lvm
  - grub
  - sysadmin
  - infrastructure
categories:
  - Linux
  - Operations
  - Virtualization
---

A few weeks ago I had to migrate a RHEL 5 server to Hyper-V.

Yes, RHEL 5.

Released in 2007.

Old enough to drive in some states.

Naturally, this server was still important. Because of course it was.

Nobody keeps legacy servers around because they are fun. They stay around because they run something important, mysterious, undocumented, or all three.

So there I was, staring at a server that had survived multiple generations of hardware, operating systems, hypervisors, managers, and probably a few organizational charts.

The plan sounded simple:

> Move the server to Hyper-V.

That sentence should come with a warning label.

<!--more-->

## Everything Looked Fine Until It Didn't

The migration itself looked normal at first.

The disk conversion completed.

The VM powered on.

The BIOS screen appeared.

GRUB loaded.

Linux started booting.

And then the server basically said:

> Nice try. Rescue mode it is.

At that point, the maintenance window got very quiet.

Every infrastructure engineer knows that silence. It is the sound of people trying not to ask, "So... how bad is it?"

## The Problem With Old Linux Migrations

The problem was not really Hyper-V.

Hyper-V was doing what Hyper-V does.

The problem was that RHEL 5 was trying to boot in a virtual hardware environment it was not originally prepared for.

Old Linux systems tend to make assumptions about:

- Storage controllers
- Disk names
- LVM availability
- Initrd contents
- Network adapters
- GRUB root devices

Those assumptions work great until you move the system somewhere else.

Then they become little landmines.

## Step 1: Boot Into Rescue Mode

The first step was to boot from the RHEL 5 ISO and enter rescue mode.

At the boot prompt:

```text
linux rescue
```

The installer may ask whether it should automatically mount the installed system.

In this case, that did not work cleanly.

That was fine.

Honestly, with RHEL 5, "that did not work cleanly" is less of a surprise and more of a lifestyle.

Eventually I got to a rescue shell:

```text
sh-3.2#
```

Now the real work could start.

## Step 2: Activate LVM Before Doing Anything Else

This was mandatory.

The installed system was using LVM, and rescue mode was not going to magically make everything available in the way I needed.

From the rescue shell, before chrooting, I ran:

```bash
lvm pvscan
lvm vgscan
lvm vgchange -ay
lvs
```

The goal was simple:

- Find the physical volumes
- Find the volume groups
- Activate the logical volumes
- Confirm the root volume was visible

Once the logical volumes showed as active, things started looking much better.

This was the first moment where I thought:

> OK. Maybe this thing is not completely dead.

Always a comforting thought during a maintenance window.

## Step 3: Mount the Installed Root Filesystem Manually

Once LVM was active, I mounted the installed root filesystem manually.

In this example, the root logical volume is genericized as `/dev/vg00/lv00`:

```bash
mkdir -p /mnt/sysimage
mount /dev/vg00/lv00 /mnt/sysimage
```

Then I checked whether it looked like a real Linux installation:

```bash
ls /mnt/sysimage
```

I wanted to see directories like:

```text
bin
boot
etc
lib
sbin
usr
var
```

When those showed up, that was a good sign.

Not victory.

Just a good sign.

With legacy systems, celebrate small wins quietly. The server can hear confidence.

## Step 4: Mount `/boot` If It Is Separate

On this system, `/boot` was separate.

Using a generic example partition:

```bash
mkdir -p /mnt/sysimage/boot
mount /dev/sda1 /mnt/sysimage/boot
```

Then I verified the boot files:

```bash
ls /mnt/sysimage/boot
```

I expected to see things like:

```text
grub
vmlinuz-*
initrd-*
```

If the mount complains that the device or mount point is busy, do not panic. Check whether it is already mounted, unmount it if needed, and mount it again cleanly:

```bash
umount /mnt/sysimage/boot
mount /dev/sda1 /mnt/sysimage/boot
```

This is one of those moments where being careful matters more than being fast.

Fast is how you turn a recoverable migration into a resume-generating event.

## Step 5: Bind-Mount Kernel Filesystems

This was the step that really mattered.

Before rebuilding the initrd, the installed system needs access to `/dev`, `/proc`, and `/sys`.

Without this, `mkinitrd` can fail with errors like:

```text
error opening /sys/block: No such file or directory
```

So before chrooting, I ran:

```bash
mount -o bind /dev  /mnt/sysimage/dev
mount -o bind /proc /mnt/sysimage/proc
mount -o bind /sys  /mnt/sysimage/sys
```

This is the kind of detail that generic migration guides love to skip.

Unfortunately, this is also the kind of detail that decides whether your server boots or whether you spend the rest of the night questioning your career choices.

## Step 6: Chroot Into the Installed System

Once the filesystems were mounted correctly, I entered the installed operating system environment:

```bash
chroot /mnt/sysimage
```

At that point, I was working inside the installed RHEL 5 system, not just the rescue environment.

That distinction matters.

A lot.

## Step 7: Identify the Installed Kernel Version

Do not blindly use `uname -r` from the rescue environment.

That can show the rescue kernel, not necessarily the installed system kernel.

Instead, I checked the installed modules:

```bash
ls -1 /lib/modules
```

I also checked `/boot`:

```bash
ls -1 /boot/vmlinuz-*
```

And confirmed what GRUB was trying to boot:

```bash
grep "^kernel" /boot/grub/grub.conf
```

In my case, the target kernel was:

```text
2.6.18-440.el5
```

Use the kernel version that actually exists on your migrated system.

Do not copy and paste kernel versions from blog posts.

Including this one.

Especially this one.

## Step 8: Make GRUB Boring

GRUB is not where I want creativity during a migration.

I reviewed:

```bash
vi /boot/grub/grub.conf
```

For this migration, I made sure the kernel line used the LVM root path directly instead of something more fragile or confusing:

```text
root=/dev/vg00/lv00
```

Example:

```text
kernel /vmlinuz-2.6.18-440.el5 ro root=/dev/vg00/lv00 rhgb quiet
```

Could UUIDs work?

Sure.

Could labels work?

Also yes.

But in this case, using the explicit LVM root path made the boot process easier to reason about while troubleshooting.

And during a maintenance window, "easy to reason about" beats "technically elegant" every time.

## Step 9: Rebuild the Initrd With Hyper-V Drivers

This was the heart of the fix.

The migrated server needed an initrd that understood the Hyper-V environment.

So I rebuilt the initrd with the Hyper-V modules explicitly included:

```bash
mkinitrd -f \
  --with=hv_vmbus \
  --with=hv_storvsc \
  --with=hv_netvsc \
  /boot/initrd-2.6.18-440.el5.img \
  2.6.18-440.el5
```

The important modules were:

- `hv_vmbus`
- `hv_storvsc`
- `hv_netvsc`

Translated into normal human language:

- Hyper-V bus support
- Hyper-V storage support
- Hyper-V network support

Without the right drivers in the initrd, the kernel may start booting but fail before it can properly access the root filesystem.

That is when you get the classic legacy Linux migration experience:

> It worked fine before we moved it.

Yes.

That is usually the point.

## Step 10: Validate the Initrd Exists

After rebuilding the initrd, I verified that the file existed:

```bash
ls -lh /boot/initrd-2.6.18-440.el5.img
```

If you want to inspect it more deeply on RHEL 5, you can unpack it temporarily:

```bash
mkdir -p /tmp/initrdcheck
cd /tmp/initrdcheck
zcat /boot/initrd-2.6.18-440.el5.img | cpio -idmv
find . -name 'hv_*.ko*' -o -name lvm
cd /
rm -rf /tmp/initrdcheck
```

This is not always necessary, but when you are troubleshooting an old system, trust but verify.

Actually, with old systems, maybe just verify.

Trust is how you get paged.

## Step 11: Exit Cleanly and Reboot

Once the initrd was rebuilt, I exited the chroot:

```bash
exit
```

Optionally, unmount everything in reverse order:

```bash
umount /mnt/sysimage/sys
umount /mnt/sysimage/proc
umount /mnt/sysimage/dev
umount /mnt/sysimage/boot
umount /mnt/sysimage
```

Then came the moment of truth:

```bash
reboot
```

Every engineer knows this part.

You watch the console.

You stop talking.

You pretend to be calm.

You are not calm.

GRUB loads.

The kernel starts.

The initrd loads.

LVM activates.

The root filesystem mounts.

And then, finally, the login prompt appears.

That login prompt is one of the most beautiful things in infrastructure.

Right up there with clean backups and change windows that end early.

## Step 12: Do Not Declare Victory Too Early

A login prompt does not mean the migration is done.

It means the operating system booted.

That is only one boss battle.

After boot, I still validated:

```bash
df -h
mount
ifconfig -a
route -n
```

Then I checked:

- Application services
- Database connectivity
- Scheduled jobs
- Mount points
- Monitoring agents
- Backup agents
- Licensing
- Anything hardcoded to old hardware

The application owner still needs to test the application.

Infrastructure can prove that the server is alive.

Only the application owner can prove that the application is actually useful.

There is a difference.

A painful difference.

## What Actually Fixed It

The winning combination was:

- Boot from RHEL 5 rescue media
- Activate LVM manually
- Mount the installed root filesystem
- Mount `/boot`
- Bind `/dev`, `/proc`, and `/sys`
- Chroot into the installed system
- Identify the real installed kernel
- Ensure GRUB points to the correct LVM root
- Rebuild the initrd with Hyper-V drivers
- Reboot and validate everything

The most important command was probably this one:

```bash
mkinitrd -f \
  --with=hv_vmbus \
  --with=hv_storvsc \
  --with=hv_netvsc \
  /boot/initrd-2.6.18-440.el5.img \
  2.6.18-440.el5
```

That is the command that made the old operating system understand enough about its new Hyper-V home to boot properly.

## Lessons Learned

The funny thing about legacy servers is that they are often legacy because they are important.

Nobody wants to upgrade them.

Nobody wants to replace them.

Nobody wants to document them.

But everybody wants them online.

So they sit there for years, quietly running critical workloads until one day someone says:

> We just need to move it.

And that is how the adventure begins.

My main lessons from this migration:

- Assume the first boot will fail.
- Confirm LVM before blaming anything else.
- Do not trust rescue mode to mount everything correctly.
- Bind-mount `/dev`, `/proc`, and `/sys` before rebuilding initrd.
- Do not use the rescue kernel version by mistake.
- Make GRUB simple.
- Explicitly include Hyper-V drivers in the initrd.
- Keep your notes clean enough that future you can understand them.

Future you is tired.

Future you is probably under pressure.

Be nice to future you.

## Final Thoughts

Would I recommend running RHEL 5 in production today?

Absolutely not.

Would I bet there are still plenty of RHEL 5 servers out there running important things?

Absolutely yes.

That is the reality of enterprise IT.

Not everything is greenfield.

Not everything is cloud-native.

Not everything has a clean Terraform module and a beautiful CI/CD pipeline.

Sometimes the job is convincing a 15-year-old Linux server to survive one more migration without throwing itself dramatically onto the floor.

And when it finally boots, you do what every experienced infrastructure engineer does.

You validate everything.

You update your notes.

You keep the rollback plan handy.

And you do not tell management it was easy.

Because it was not easy.

You just made it look that way.

---

## Quick Command Reference

For the impatient future version of me:

```bash
# From rescue shell
lvm pvscan
lvm vgscan
lvm vgchange -ay
lvs

mkdir -p /mnt/sysimage
mount /dev/vg00/lv00 /mnt/sysimage

mkdir -p /mnt/sysimage/boot
mount /dev/sda1 /mnt/sysimage/boot

mount -o bind /dev  /mnt/sysimage/dev
mount -o bind /proc /mnt/sysimage/proc
mount -o bind /sys  /mnt/sysimage/sys

chroot /mnt/sysimage

ls -1 /lib/modules
ls -1 /boot/vmlinuz-*
grep "^kernel" /boot/grub/grub.conf

vi /boot/grub/grub.conf

mkinitrd -f \
  --with=hv_vmbus \
  --with=hv_storvsc \
  --with=hv_netvsc \
  /boot/initrd-2.6.18-440.el5.img \
  2.6.18-440.el5

ls -lh /boot/initrd-2.6.18-440.el5.img

exit

umount /mnt/sysimage/sys
umount /mnt/sysimage/proc
umount /mnt/sysimage/dev
umount /mnt/sysimage/boot
umount /mnt/sysimage

reboot
```

---

## About Ops Under Pressure

Ops Under Pressure is a collection of real-world infrastructure stories, troubleshooting adventures, automation projects, and lessons learned from the trenches of enterprise IT.

Because the most interesting part of the job starts right after someone says:

> This should be easy.
