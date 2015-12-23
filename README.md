# ArchLinux installation scripts

RHEL has kickstart, Debian has preseed, Windows has autounattend.xml, and Arch has...
bash? Works for me!

These are some helper scripts to install a basic ArchLinux system from a plain archiso,
with a minimum of manual input. Goal is to boot archiso, connect to network, type one
command, optionally change some defaults such as static IP or root password, and then
have a complete system, according to install guide. At that point, the system is ready
for a configuration management tool to finish the job.

## Features

This is what it can do:

- GPT and MBR partitions
- LUKS encryption with encrypted boot and keyfiles in initramfs
- Use grub2 as boot loader (others could be added)
- Configure timezone, locale, netctl, systemd services
- Custom post install steps

## Preparation

Write a script that exports two or three bash functions and some variables, and then
curl-executes another script that partitions, formats, installs and configures a basic
Arch system.

See gh-pages branch for some examples from my home network.

## Run the installer

First, boot the arch ISO. Connect to network using `wifi-menu` if needed. Then, to run
the script:

    { curl -s j0057.github.io/arch/[hostname] | bash -e } 4<&0

So curl downloads the script, which is piped over stdin to bash. You'll be prompted for
every variable that starts with "INSTALL\_", but since stdin is already used for the stdout
of curl, the input for the prompts must be entered through file descriptor 4.

## Host scripts

See the gh-pages branch for some examples from my home network.

### Functions

There are some helper functions are defined in `arch_fn`.

#### Callback: configure\_disk

This function should be exported with `export -f`, and it should partition, format and mount
the disk, ideally using `partition()`.

#### Callback: configure\_network

This function should configure the network... whatever that means. For me, it means tying the
MAC addresses to interface names using udev rules, creating netctl profiles, and later 
enabling `netctl-ifplugd@.service` for ethernet interfaces, and `netctl-auto@.service` for
wireless interfaces.

TODO: create a `network()` helper that does this

#### Callback: post\_install

This function can do any essential post-installation configuration steps inside the stage 2
chroot. 

For example, I have this old laptop that will hibernate every 15-20 seconds, because the lid
switch is flaky, so systemd should ignore the lid switch signal. :-)

#### Helper: partition

Partitions, encrypts, formats and mounts a disk.

Usage: `partition TARGET TYPE [DESC...]`

Where TARGET is the disk to install on, such as `/dev/sda`, TYPE is either `mbr` or `gpt`, and DESC
is one or more positional arguments, where each argument is a string that has multiple whitespace
separated values. The fields are:

* num: The index number of the partition
* size: An offset:size combination as understood by `sgdisk` (see manpage)
* typecode: Type of the partition
* label: Label for the partition (if GPT) and label for the filesystem (MBR and GPT)
* fstype: Filesystem type
* mountpoint: Where to mount the filesystem, or `-` for swap

The partitions are created in the order as described by the `num` field, but mounted in the order
given to the `partition` function, that is, the root partition should be mounted before a `/home` or
a `/boot` partition

#### Helper: line

Adds one or more lines to a file.

Usage: `lines TARGET [LINE...]`

Where TARGET is the file to write to, and LINE is one or more strings that should each be added as
a line in the target file.

### Variables

These variables are used in all the scripts.

#### INSTALL\_HOST

Base URL for downloading scripts, for example `"j0057.github.io/arch"` or `"some-other-host:1234"`.
Will be used to construct URLs for the different stages of the install scripts.

#### INSTALL\_HOSTNAME

Hostname for the system.

#### INSTALL\_TZNAME

Timezone name, for example `"Europe/Amsterdam"`

#### INSTALL\_LOCALE

Locale config

#### INSTALL\_PACKAGES

The packages to install, should minimally include `base` and `grub`.

#### INSTALL\_SERVICES

Which systemd units to enable. Can handle `.target` units too, in that case the default target is set.

#### INSTALL\_PASSWORD

The password to use for the root account and for decrypting /boot.

## Stages

The per-host script downloads and runs arch\_stage0, which does the following things:

1. For each install variable, prompt the user to override the value;
2. Download and source arch\_fn, which contains helper functions;
3. Download and run arch\_stage1, which partitions, encrypts and formats the disk,
   and uses `pacstrap` to install the system – basically, all the things that happen
   outside the chroot
3. Download and run arch\_stage2, which configures the timezone, root password,
   initramfs, boot loader, the network and so on
4. Unmount the target file systems

## Developing/branch strategy

The per-hosts scripts can be started from a gh-pages branch that's kept rebased on top of the `master`
branch on `github.com:j0057/arch`. That means they can be reached at `http://[ghuser].github.io/arch/[hostname]`.

When developing, it's easier to just use `python2 -m SimpleHTTPServer 12345` from a local development
server, and start the script like this:

    { curl -s somewhere.lan:12345/[hostname] | bash -es somewhere.lan:12345 } 4<&0

This way, using the `-s` parameter to bash, `INSTALL_HOST` can set the default value to
`somewhere.lan:12345` instead of the default in the script, and you still only have to mash enter a few
times to to test the installation.
