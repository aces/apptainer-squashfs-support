
# SquashFS through Apptainer : Hints And Tips

This repository contains code, examples, hints and other documentation
related to using Apptainer containers as access methods to overlay
files (typically, squashfs files).

The data organization addressed here generally consists of

1. Having one or several `.squashfs` files that contain the data;
2. Having an Apptainer container image with specific capabilities (`rsync`, `openssh` etc);
3. Combining the two, along with other scripts, to build a system that can seemlessly access the data files.

## What this repository contains

* The directory `build_data` contains code and instructions to make squashfs files;
* The directory `build_simg` contains code and instructions to build an Apptainer container;
* The directory `bin` contain some utility scripts (e.g. `apptainer_sftpd`);
* The directory `doc_examples` contains sample README files to install with your data, for helping users access it;
* The directory `images` contains a PDF with technical diagrams, and its source in OmniGraffle format;
* The rest of this README here contains hints and code snippets on accessing the data files.

Note: this repository is a newer version of [sing-squashfs-support](https://github.com/aces/sing-squashfs-support)
with the nomenclature adjusted to refer to `Apptainer` instead of `Singularity`, and better
support for files with extensions `.sqfs`.

## Accessing the data files

For the examples below, let's assume we have a data distribution
directory called `/data/HCPsquash` containing two SquashFS filesystem
files, and an Apptainer container image file:

```bash
unix% ls -l /data/HCPsquash
total 83068518941
-rw-r--r-- 1 prioux rpp-aevans-ab 1508677619712 Aug  1 16:13 hcp1200-00-100206-103414.squashfs
-rw-r--r-- 1 prioux rpp-aevans-ab 1532533477376 Aug  1 20:33 hcp1200-01-103515-108020.squashfs
-rwxr-xr-x 1 prioux rpp-aevans-ab     147062784 Dec  3 16:51 apptainer_squashfs.sif
```

(This example is taken as a subset of a real dataset, and more
information about it can be found by reading the file
[README.txt](doc_examples/hcp_1200_README.txt) that was provided to its
users)

The two squashfs files store a bunch of data files inside them under
the root path `/HCP_1200_data`. The first file contains 20
subdirectories named `100206` ... `103414`, and the second file
contains 20 subdirectories named `103515` ... `108020`.

**Important note:** Some versions of Apptainer are known to
require a suffix consisting of the three characters `:ro` after the
names of the overlays; the commands below would, for instance,
require all overlay options to be in the form of `--overlay=abc.squashfs:ro` .

### a) Connecting interactively (low-level, directly)

This will allow you to have a look at the files, with only the first
squashfs file mounted:

```bash
cd /data/HCPsquash
apptainer shell --overlay=hcp1200-00-100206-103414.squashfs apptainer_squashfs.sif
```

You can then `cd /HCP_1200_data` and `ls` the files. Use `exit` to
exit the container!

To get both squashfs files:

```bash
apptainer shell --overlay=hcp1200-00-100206-103414.squashfs --overlay=hcp1200-01-103515-108020.squashfs apptainer_squashfs.sif
```

Now you can notice that the content of `/HCP_1200_data` has 40
subdirectories instead of just 20.

To connect with all .squashfs file, no matter how many:

```bash
apptainer shell $(ls -1 | grep '\.squashfs$' | sed -e 's/^/--overlay /') apptainer_squashfs.sif
```

To disable the messages about the squashfs not being a writable
filesystem, use the `-s` option of Apptainer:

```bash
apptainer -s shell ...
```

### b) Running a command (low-level, directly)

This is just like in a) above, but instead of running `apptainer shell`
we run `apptainer exec`:

```bash
apptainer -s exec --overlay=hcp1200-00-100206-103414.squashfs apptainer_squashfs.sif ls -l /HCP_1200_data
```

### c) Running a command or a shell (with utility wrapper)

In the [bin](bin) directory of this repo, you will find a set of
utility wrapper scripts. In fact, it's a single script with multiple
names. It has many features and options allowing you to choose which
squashfs files to access and which Apptainer image file to run,
but the simplest use scenario is to copy the one called `apptainer_command_here`
into the same directory `/data/HCPsquash` as the squashfs files and
Apptainer image:

```
unix% ls -l /data/HCPsquash
total 83068518941
-rw-r--r-- 1 prioux rpp-aevans-ab 1508677619712 Aug  1 16:13 hcp1200-00-100206-103414.squashfs
-rw-r--r-- 1 prioux rpp-aevans-ab 1532533477376 Aug  1 20:33 hcp1200-01-103515-108020.squashfs
-rwxr-xr-x 3 prioux rpp-aevans-ab          7542 Dec  3 16:57 apptainer_command_here
-rwxr-xr-x 1 prioux rpp-aevans-ab     147062784 Dec  3 16:51 apptainer_squashfs.sif
```

When invoked, it will automatically detect those files around it,
and run a `apptainer exec` command with all the appropriate
overlays. Now you can run the same command as in example b) above,
but in a simpler way:

```bash
# Run on all squashfs files:
./apptainer_command_here ls -l /HCP_1200_data

# Run on just one squashfs file:
./apptainer_command_here -O hcp1200-00-100206-103414.squashfs ls -l /HCP_1200_data

# Connect interactively:
./apptainer_shell_here -O hcp1200-00-100206-103414.squashfs # with one data file
./apptainer_shell_here                                      # with all files
```

### d) Mounting the data files using sshfs

Running programs from within the container is the most efficient
option for accessing the data files. If that option is not available,
or if the data files need to be accessed remotely, then it is also
possible to mount the data directory using sshfs, from elsewhere.

Note that mounting the data with sshfs impose a significant performance
penalty, as encryption and decryption of the SFTP traffic (which
transports the mountpoint's data files) will occur at all times.

The first thing to realize is that we can't simply use a normal
sshfs mount command, because the container is not initially started.
Even if the container is started, it:

* doesn't run a sshd deamon;
* it is not even addressable with a network address.

In [Diagram #2](images/diag_02_SSHFS.png) we can see that a normal
sshfs mount results in the program `sftp-server` to be launched on
the remote site. This program is connected through its stdin and
stdout channels to the FUSE client on the local site. Filesystem
operations within the mount point (open, read, seek etc) are
translated into SFTP operations sent through the ssh connection to
that remote `sftp-server` program.

What we need to do is tell the sshfs mounter to launch, on the
remote site, its client `sftp-server` **inside** the container. It will
run as a standalone normal process, even though it will still talk
to its launcher sshd program through stdin and stdout. A very
complicated way of doing this would be to mount the filesystem with:

```bash
# Complicated example; do NOT DO this!

# Moutpoint: an empty dir
mkdir mymountpoint

# See how complicated the sftp_server command is, and we're
# using just ONE of the overlays too!
sshfs -o sftp_server="apptainer -s exec --overlay=/data/HCPsquash/hcp1200-00-100206-103414.squashfs /data/HCPsquash/apptainer_squashfs.sif /usr/libexec/openssh/sftp-server" user@computer2:/HCP_1200_data mymountpoint

ls mymountpoint
fusermount -u mymountpoint
```

The sshfs option `-o sftp_server=` is rare and unusal. It is not
normally required with scp or sshfs, as there are no real alternative
compatible sftp servers other than the one that comes with the
OpenSSH package.

A better way of performing the same thing without having to provide
a long command to the `-o sftp_server=` option is to first pack
that long command into a separate bash script. Let's call it
`example1.sh` :

```bash
#!/bin/bash

# Content of example1.sh

apptainer -s exec \
  --overlay=/data/HCPsquash/hcp1200-00-100206-103414.squashfs \
  --overlay=/data/HCPsquash/hcp1200-01-103515-108020.squashfs \
  /data/HCPsquash/apptainer_squashfs.sif                          \
  /usr/libexec/openssh/sftp-server
```

Then the mount command becomes a much simpler:

```bash
sshfs -o sftp_server="/path/to/example1.sh" user@computer2:/HCP_1200_data mymountpoint
```

We can have another look at this solution in [Diagram #3](images/diag_03_SSHFS_Setup.png)
and [Diagram #4](images/diag_04_SSHFS_SINGSFTPD.png).

This will work fine as long as the content of `example1.sh` is
updated appropriately whenever the Apptainer container is changed,
or the set of overlays are changed.

A better solution would be to create a new shell wrapper that
works like `example1.sh` but in a more generic way. The [bin](bin)
directory in this repo contains such programs, `apptainer_sftpd` and
`apptainer_sftpd_here`.  They come with full documentation, just run
them with the `-h` option. But in essence, installing `apptainer_sftpd_here`
in the `/data/HCPsquash` directory will make it automatically detect
all the `.squashfs` and the `.sif` file there, and allow you to
mount the data files with:

```bash
sshfs -o sftp_server="/data/HCPsquash/apptainer_sftpd_here" user@computer2:/HCP_1200_data mymountpoint
```

### e) Copying the data files using scp

Just like for sshfs above, it is possible to run the scp command's
server-side program with an alternative SFTP server. The option
is in fact exactly the same:

```bash
scp -o sftp_server="apptainer -s exec --overlay=/data/HCPsquash/hcp1200-00-100206-103414.squashfs /data/HCPsquash/apptainer_squashfs.sif /usr/libexec/openssh/sftp-server" user@computer2:/HCP_1200_data/remote_file.txt localfile.txt
```

or more simply using the same type of wrapper described for sshfs:

```bash
scp -o sftp_server="/data/HCPsquash/apptainer_sftpd_here" user@computer2:/HCP_1200_data/remote_file.txt localfile.txt
```

### f) Extracting data using rsync

Just like for sshfs in section d) above, we can't simply rsync
the data files out of the `.squashfs` files from the outside if the
rsync program runs on the host where these squashfs files reside.
[Diagram #5](images/diag_05_RSYNC.png) shows the architecture of a
standard rsync session. The rsync program running on computer2 would
be outside of a proper container.

But just like for sshfs, we can also fix that. The rsync program
support an option `--rsync-path=/abc/def/prog` and so if we provide
some `/abc/def/prog` that acts like a rsync program, the architecture
is respected. The way to do that is once again to create a bash
wrapper `example2.sh`:

```bash
#!/bin/bash

# Content of example2.sh

apptainer -s exec \
  --overlay=/data/HCPsquash/hcp1200-00-100206-103414.squashfs \
  --overlay=/data/HCPsquash/hcp1200-01-103515-108020.squashfs \
  /data/HCPsquash/apptainer_squashfs.sif                          \
  rsync "$@"
```

It is now possible to rsync data out of the `.squashfs` files in
this way:

```bash
rsync -a --rsync-path="/path/to/example2.sh" user@computer2:/HCP_1200_data/123456 ./123456_copy
```

This solution is shown in [Diagram #6](images/diag_06_RSYNC_Setup.png)
and [Diagram #7](images/diag_07_RSYNC_SINGRSYNC.png).

Again, a more general solution is provided in the [bin](bin) directory
of this repo, where you can find two utilities named `apptainer_rsync`
and `apptainer_rsync_here`. These can be deployed alongside the `.squashfs`
filesystem files and the Apptainer image to make the process of
recognizing them and booting the Apptainer container transparent.
Running `apptainer_rsync` with the `-h` option will provide more
information about these utilties.

## Other tricks and tips

### Writable overlay

Files in `.squashfs` format encode *read-only* filesystems. They
are perfect for large static datasets, as they are very fast and
reduce tremendously the inode requirements on the host filesystem.

While a process is running inside an Apptainer container, that
process can write files only on externally mounted writable
filesystems; Apptainer normally provides /tmp and the $HOME
directory of the user who runs the apptainer command. Other mount
point can be provided by adding explicit `-B` options to the apptainer
command line too.

The utility programs included in [bin](bin) will launch Apptainer
containers with not only all the `.squashfs` that they can find,
but also any file with a `.ext3` extension. These can be built as
formatted EXT3 filesystem and apptainer will make them writable.
For more information about building such files, consult the repo
for the utility [withoverlay](https://github.com/prioux/withoverlay).

