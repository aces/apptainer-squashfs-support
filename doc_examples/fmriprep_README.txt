
##########################################################
# UK BioBank fMRIPrep Outputs Release in SquashFS format #
##########################################################

This directory contains the outputs of the fMRIPrep pipeline run over 37732
subjects from the UK Biobank dataset.

It contains 16,935,185 files and uses 117 terabytes of disk space.

The outputs are packed in 189 SquashFS files named in this way:

  fmriprep_000_1000011-1025826.sqfs
  fmriprep_001_1025941-1049845.sqfs
  ... (186 other files)
  fmriprep_188_6010980-6024935.sqfs

In the first '000' file, the subjects IDs 1000011 to 1025826 are present.
Each SquashFS file contains 200 subjects, except for the last one which
contains 132.

Because the amount of data and the number of files are so large, the
NeuroHub team is asking all scientists not to make full copies of these
SquashFS files. Instead, there are multiple ways to access the contents of
the SquashFS files directly. The following sections provide examples on how
to browse the fMRIPrep outputs interactively, and how to provide them to
tools and scripts for processing non-interactively.


========================================================================
Table of content:
========================================================================
1) General notes about Apptainer and containerization
2) Accessing the files interactively
3) Running scientific tools within containers
4) Extracting files
5) Mounting the files with SSHFS
6) More help and hints
7) Credits


========================================================================
1) General notes about Apptainer and containerization
========================================================================

Apptainer is a linux technology to launch 'containers'. A container is a
sort of independent namespace for files and processes, but unlike a virtual
machine, the container is running on the same computer kernel where it was
launched.

Apptainer containers are often an easy way to package together a scientific
application, including all the necessary libraries and static data files,
saving the user the steps of installing the application manually. A
container's code will often present itself as a single large file,
typically named 'something.sif'. These files can also be built
automatically out of Docker containers (another containerization
technology).

When an Apptainer container is launched, the user can specify that some
SquashFS files be 'mounted' within the container, so that the files encoded
in those SquashFS files become visible as normal files to all of the
container's processes.

Containers are started with the 'apptainer' command. In this document we
will only discuss two modes of operation, which are specified by two
sub-commands. 'apptainer shell' starts an interactive shell in a container,
and 'apptainer exec' starts a command in a container. An 'apptainer shell'
session last as long as the user types command, until the user exits. An
'apptainer exec' session lasts until the command finishes.

In all the examples below, the container's code we will use is found in the
file named 'apptainer_squashfs.sif'. This file is located alongside the
fMRIPrep SquashFS files. It is a demonstration container with a rather
standard linux distribution.


========================================================================
2) Accessing the files interactively
========================================================================

As a first exercise, you might want to examine a plain container with no
SquashFS data file at all. This can be accomplished with the following
commands:

  cd /project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep
  apptainer shell apptainer_squashfs.sif

You'll get a prompt that looks like 'Apptainer>'. You'll be in your beluga
home directory (provided as a convenience by Apptainer), but all files
outside your home directory are provided by the container itself. If
you're familiar with Linux you should be able to discover that the
container is a Ubuntu 22.04.3 LTS distribution. Type 'exit' to return to
your shell on beluga.

To start a containerized shell and get access to the fMRIPrep outputs,
you'll need to tell Apptainer to mount the SquashFS files. This is
accomplished by configuring the SquashFS file as an 'overlay'. Let's try
it with the first SquashFS file, 'fmriprep_000_1000011-1025826.sqfs':

  apptainer shell --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif

This time, you'll find some of the fMRIPrep output files appearing under
this containerized directory: /neurohub/ukbb/derivatives/fMRIPrep

  # Within the container:
  cd /neurohub/ukbb/derivatives/fMRIPrep
  ls     # you should see 200 fMRIPrep outputs
  exit   # return to beluga

To get the first 400 fMRIPrep results (in the first two SquashFS file), you
would need to add another '--overlay=' option:

  # Note: this is a single long line
  apptainer shell --overlay=fmriprep_000_1000011-1025826.sqfs --overlay=fmriprep_001_1025941-1049845.sqfs apptainer_squashfs.sif

This can be scaled up to about 50 overlays. You will probably not be able
to start a container with all 189 overlays, as there are system
restrictions that will prevent that.

Section 6 below contains helpful hints and tricks about running such
commands at large scale, and about accessing other directories within the
container too.


========================================================================
3) Running scientific tools within containers
========================================================================

The previous section showed you how to start the container stored in the
file 'apptainer_squashfs.sif'. As explained, this container is just a plain
Ubuntu operating system with very few things aside from the base OS. We
provide it as an example container.

If you are planning to run a scientific tool on the fMRIPrep outputs, it's
best to get the version of the tool that is also containerized. In that
way, when you start your analysis, you'll only have to add the options to
mount the SquashFS files and your tool will see all the fMRIPrep output
files naturally.

The examples below will still use our demonstration
'apptainer_squashfs.sif' container, and we'll be running standard linux
utilities 'ls' and 'du' as the "tool".

Starting a containerized 'my_command' within a container is performed by
providing my_command to 'appainter exec'. The general format is:

  apptainer exec [apptainer options] container_image my_command

For example, the 'ls' command of section 2 above can be run with:

  apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif ls /neurohub/ukbb/derivatives/fMRIPrep

This boots the container, mounting the fMRIPrep outputs, and then runs the
'ls' command. Once the 'ls' command finishes, the container disappears and
you are back to beluga.

A set of several commands need to be started with 'bash -c' like this:

  apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif bash -c "cd /neurohub/ukbb/derivatives/fMRIPrep;du -h *0114"

Or, an entire script can be provided in standard input with this type of
setup:

  # As a redirection
  apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif bash < script.sh
  # As a pipe
  cat script.sh | apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif bash

Section 6 below contains helpful hints and tricks about running such
commands at large scale, and about accessing other directories within the
container too.


========================================================================
4) Extracting files
========================================================================

Do not attempt to extract all the files. There are just too many. But if
you need a specific set of files out of each subject, it might become
manageable. There are several ways of doing this.

One way is to use an interactive shell like described at section 2 above.
You enter the container and use the 'cp' command to copy your files to some
other place.

  # Start a container shell
  apptainer shell --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif

  # (In the container shell, copy a file to my home directory)
  cd /neurohub/ukbb/derivatives/fMRIPrep/sub-1010114
  cd sub-1010114/ses-2/func
  cp sub-1010114_ses-2_task-rest_boldref.nii.gz $HOME
  exit # return to beluga

A second way is to run a command (such as cp or rsync) on the Apptainer
command line, like explained in section 3 above:

  # Run a cp command directly
  apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif cp /neurohub/ukbb/derivatives/fMRIPrep/sub-1010114/sub-1010114/ses-2/func/sub-1010114_ses-2_task-rest_boldref.nii.gz $HOME

  # Run a rsync command directly to copy all func/*nii.gz of subject 1000011
  apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif rsync -a "/neurohub/ukbb/derivatives/fMRIPrep/sub-1010114/sub-1010114/ses-2/func/*.nii.gz" $HOME

The third and most sophisticated way is to run an 'rsync' server
containerized while the rsync client is running on beluga. This is quite
tricky, but it can provide a way to pull files from any other computer that
can connect to beluga.

  # These two variables are used to make the command shorter, below
  datapath="/project/6008063/neurohub/ukbb/derivatives/fmriprep"
  rsync_in_appt="apptainer exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif rsync"
  # Copy the full func/ subdirectory to my home
  rsync -a -i --rsync-path="cd $datapath;$rsync_in_appt" $USER@localhost:/neurohub/ukbb/derivatives/fMRIPrep/sub-1010114/sub-1010114/ses-2/func/ ~/func

This mechanism, unlike the other two, allows you to fetch the file contents
from the SquashFS file while you are on beluga if you connect as
'$USER@localhost:' (as shown in the example), but also from outside of
beluga if you connect TO beluga using 'username@beluga:'

Section 6 below contains helpful hints and tricks about running such
commands at large scale, and about accessing other directories within the
container too.


========================================================================
5) Mounting the files with SSHFS
========================================================================

If you need to access the fMRIPrep files from programs running outside of a
container (for instance, if your program is a matlab or python script that
is not available in a container), then you can create what is called a
SSHFS mount point for them. The basic steps are:

a. creating an empty directory somewhere;
b. starting in the background a special SSHFS process that maps the fMRIPrep output
files in it;
c. running your program so that it access the files newly visible in the
directory;
d. stopping the SSHFS background process when you are done.

This can work locally on beluga, or on a workstation outside of beluga
(e.g. your lab's computer or your personal computer). However, note that
this mechanism will make the files visible only on the computer that is
running the SSHFS program. On a distributed file system like on beluga, you
cannot mount the files on (say) beluga1 and expect to see the fMRIPrep
files from beluga2 or the compute nodes.

Here is a short example on how to mount the first fMRIPrep output into the
empty directory '/tmp/fprep_000_$USER' (where $USER will be replaced by
your beluga username). In this example we mount the files on a computer
already on beluga, and expect to access them from there too.

  # These four variables are used to make the command shorter, below
  squashfiles="/project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep"
  allsubjects="/neurohub/ukbb/derivatives/fMRIPrep"
  sftp_server="/usr/lib/openssh/sftp-server"
  sftp_server_command="cd $squashfiles;apptainer -s exec --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif $sftp_server"

  # Create empty directory as mount point
  mkdir -p /tmp/fprep_000_$USER
  chmod 755 /tmp/fprep_000_$USER

  # Start the SSHFS server connected to the container
  sshfs -o sftp_server="$sftp_server_command" $USER@localhost:$allsubjects /tmp/fprep_000_$USER

  # Access the data at will, and then unmount the data when you're done.
  ls /tmp/fprep_000_$USER
  fusermount -u /tmp/fprep_000_$USER

A few important notes:

a. Our example creates a mountpoint in /tmp because it is the easiest way
to demonstrate the architecture. It is possible to create the mount point
elsewhere, but be warned that adjustments need to be made to file
permissions and the APPTAINER_BINDPATH settings for this to work. See
section 6 for more information.

b. If you are creating your mountpoint in /tmp like in this example, you
should use a name specific to you, given /tmp is also visible and shared by
all users. Don't try to create a directory '/tmp/fprep' because it could
already exist and be in use by someone else. That is why in our example, we
chose to create a directory name that contains your username in it
(fprep_000_$USER).


========================================================================
6) More help and hints
========================================================================

a) Utilities
------------

The examples of section 2, 3, 4 and 5, above, all involve at some point
running the 'apptainer' command with either the 'shell' or 'exec'
subcommands, a bunch of '--overlay' options, and the name of a container.
For instance, as shown in section 2, an interactive shell to examine the
content of the first SquashFS file is launched with this command:

  apptainer shell --overlay=fmriprep_000_1000011-1025826.sqfs apptainer_squashfs.sif

These commands can quickly become unwieldy when several overlays need to be
configured. Each new overlay adds an extra --overlay=squashfsfile to the
command.

This is where helper commands or utilities can make your job easier. In
the same directory as the fMRIPrep output files, there are four utility
programs we created for you:

  apptainer_command
  apptainer_rsync
  apptainer_sftpd
  apptainer_shell

(The variations that have _here appended will be explained later. Also, do
not confuse 'apptainer_shell' with the command 'apptainer shell')

These commands simplify the use of Apptainer with multiple SquashFS files.
They all take three options:

   -D directory_path
   -I apptainer_image_file
   -O squashfile_pattern

The -D option tells these commands which directory contains both the
SquashFS files and the Apptainer image file. You can (optionally) specify
the name of the Apptainer image file to use for the container (with -I),
and which SquashFS files to use as overlays (with -O). For instance,
instead of typing the long 'apptainer shell' command above shown at the
beginning of this section, you can run:

  cd /project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep
  ./apptainer_shell -D $PWD -O fmriprep_000_1000011-1025826.sqfs

That tells the utility apptainer_shell to use the current directory ($PWD)
for finding all the files and only use the first SquashFS file as the
overlay. It will detect and automatically use the file
"apptainer_squashfs.sif" for the name of the container (but you could
override this with the -I option).

That may not seem like a big improvement in the length of the command, but
let's see what happens when we take advantage of the utility's ability to
do pattern matching. You want to start a shell with the first 20 SquashFS
files? Normally that would be a very long command to type, but the utility
does it for you:

  cd /project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep
  ./apptainer_shell -D $PWD -O 'fmriprep_0[01]*'

The fact we gave the program a pattern that expands to the first 20
SquashFS files means the Apptainer shell should have access to 4000
subjects (20 x 200) in your session under
/neurohub/ukbb/derivatives/fMRIPrep.

Now, what about these special '_here' versions of these commands?  These
are meant to simplify your life even further: when you run them, you no
longer have to provide the '-D' option. The utility program will implicitly
use the directory where the program itself resides. This works even if you
copy the script elsewhere, or run the script where your current Directory
is not the place where the SquashFS file are!

  # When in the location of the SquashFS files:
  ./apptainer_shell_here -O 'fmriprep_00*'  # browse the first 10 SquashFS
  # From your home directory
  cd $HOME
  /project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep/apptainer_shell_here -O 'fmriprep_00*'

The other utility scripts provide shortcuts for the commands described in
the other sections. For instance:

  # Running the ls command like in section 3 above on 10 SquashFS files
  ./apptainer_command_here -O 'fmriprep_00*' ls /neurohub/ukbb/derivatives/fMRIPrep

  # Extracting files like in section 4 above. Notice that we no longer
  # have to do a 'cd' as part of the rsync-path option:
  rsync -a -i --rsync-path="/project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep/apptainer_rsync_here -O 'fmriprep_000*'" $USER@localhost:/neurohub/ukbb/derivatives/fMRIPrep/sub-1010114/sub-1010114/ses-2/func/ ~/func

  # Mounting a SSHFS directory like in section 5 above. Again,we no longer need a 'cd':
  sshfs -o sftp_server="/project/rpp-aevans-ab/neurohub/ukbb/derivatives/fmriprep/apptainer_sftpd_here -O 'fmriprep_000*'" $USER@localhost:$allsubjects /tmp/fprep_000_$USER

A whole lot of other options and architecture configurations are described
in the main GitHub repo for these utilities, found at:

  https://github.com/aces/apptainer-squashfs-support

Don't hesitate to contact the NeuroHub team if you need to set up a
particular system that's not covered in these solutions here. See the
Credits section below for more help.

b) Mounting other directories
-----------------------------

Note that Apptainer, by default, only mounts your home directory in the
container. To be able to access files outside of the home directory (such
as files in the project space), you'll need to tell the apptainer command
to mount them. There are two ways to do so: with the environment variable
APPTAINER_BINDPATH or with the '-B' option.

For instance, let's say you'd like to have access to

   /scratch/$USER

and

   /lustre03/project/rpp-aevans-ab/$USER

then your Apptainer commands could start with one of these ways:

   # Using an environment variable exported within your shell
   export APPTAINER_BINDPATH=/scratch/$USER,/lustre03/project/rpp-aevans-ab/$USER
   apptainer shell --overlay etc etc

   # Using an environment variable exported just to the apptainer command
   APPTAINER_BINDPATH=/scratch/$USER,/lustre03/project/rpp-aevans-ab/$USER apptainer shell --overlay etc etc

   # Using the -B option
   apptainer shell -B /scratch/$USER,/lustre03/project/rpp-aevans-ab/$USER --overlay etc etc

Note that the environment variable solutions are the only ones available to
you if you are using the utility programs described in section 6-a above,
since you do not have a way to set Apptainer's '-B' option within those
scripts.


========================================================================
7) Credits
========================================================================

The dataset was prepared as a mandate by the NeuroHub Project.
The institutions and research groups involved are:

a) The UK BioBank Project
b) McGill University
c) The Montreal Neurological Institute
d) Alan Evan's Lab at the MNI and McGill
e) The McGill Center For Integrative Neuroscience
f) Healthy Brains For Healthy Lives
g) The NeuroHub project
h) The CBRAIN Project

This document was written by:
  Pierre Rioux <pierre.rioux@mcgill.ca> from the CBRAIN Project, and
  Darcy Quesnel <darcy.quesnel@calculquebec.ca> from Calcul Quebec.

For support with these files and commands, contact <support@neurohub.ca>.

