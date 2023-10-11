
# How to build a container image file suitable for accessing squashfs files

Accessing SquashFS files as a non-privileged user is accomplished
through a Apptainer container. The image for that container doesn't
need to include anything particular if one just wants to enter it
interactively and work from within the container.

If the container is to be used by other external access programs
(e.g. sshfs or rsync, as explained in other parts of this GitHub repo)
then it must include a few more system packages:

* openssh-server :
  This package provides /usr/libexec/openssh/sftp-server (Centos) or /usr/lib/sftp-server (Ubuntu)
* rsync :
  This package provides the rsync executable

To build an apptainer container image with both of these, we provide
here a [apptainer definition file](https://apptainer.org/docs/user/main/definition_files.html)
called `apptainer_squashfs.def`.

This command (run as root) will build the container in the file `apptainer_squashfs.sif`.

```shell
apptainer build apptainer_squashfs.sif apptainer_squashfs.def
```

Note that apptainer doesn't care all that much about the extension given
to the image file, so you can also call it with `.simg` or `.img` instead of `.sif`.

