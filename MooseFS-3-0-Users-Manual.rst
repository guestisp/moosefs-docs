| |image|
|  **MooseFS 3.0 User’s Manual
  **

Core Technology Development & Support Team

| Copyright 2014- v. 1.0.6
| Piotr Robert Konopelko, Core Technology Development & Support Team.

| *Proofread by* Agata Kruszona-Zawadzka
| *Coordination & layout by* Piotr Robert Konopelko.

Please send corrections to `Piotr Robert
Konopelko <mailto:peter@mfs.io>`__ – peter@mfs.io.

This file is part of MooseFS.

MooseFS is free software; you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free
Software Foundation, version 2 (only).

MooseFS is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for
more details.

You should have received a copy of the GNU General Public License along
with MooseFS; if not, write to the Free Software Foundation, Inc., 51
Franklin St, Fifth Floor, Boston, MA 02111-1301, USA or visit
http://www.gnu.org/licenses/gpl-2.0.html

About MooseFS
=============

MooseFS is a fault-tolerant distributed file system. It spreads data
over several physical locations (servers), which are visible to user as
one resource. For standard file operations MooseFS acts as any other
Unix-alike filesystem:

-  Hierarchical structure (directory tree)

-  Stores POSIX file attributes (permissions, last access and
   modification times)

-  Supports special files (block and character devices, pipes and
   sockets)

-  Symbolic links (file names pointing to target files, not necessarily
   on MooseFS) and hard links (different names of files that refer to
   the same data on MooseFS)

-  Access to the file system can be limited based on IP address and/or
   password

Distinctive features of MooseFS are:

-  High reliability (several copies of the data can be stored on
   separate physical machines)

-  Capacity is dynamically expandable by adding new computers/disks

-  Deleted files are retained for a configurable period of time (a file
   system level “trash bin”)

-  Coherent snapshots of files, even while the file is being
   written/accessed

Architecture
------------

MooseFS consists of four components:

#. Managing servers (s) – In MooseFS one machine, in MooseFS Pro any
   number of machines managing the whole filesystem, storing metadata
   for every file (information on size, attributes and file location(s),
   including all information about non-regular files, i.e. directories,
   sockets, pipes and devices).

#. Data servers () – any number of commodity servers storing files’ data
   and synchronizing it among themselves (if a certain file is supposed
   to exist in more than one copy).

#. Metadata backup server(s) () – any number of servers, all of which
   store metadata changelogs and periodically download main metadata
   file.

   In MooseFS (non-Pro) machine with Metalogger can be easily set up as
   a master in case of main master failure.

   In MooseFS Pro Metalogger can be set up to provide an additional
   level of security.

#. Client computers that access () the files in MooseFS – any number of
   machines using process to communicate with the managing server (to
   receive and modify file metadata) and with chunkservers (to exchange
   actual file data).

is based on the FUSE [1]_ mechanism (Filesystem in USErspace), so
MooseFS is available on every Operating System with a working FUSE
implementation (Linux, FreeBSD, MacOS X, etc.)

| |image|

| |image|

Metadata is stored in the memory of the managing server and
simultaneously saved to disk (as a periodically updated binary file and
immediately updated incremental logs). The main binary file as well as
the logs are synchronized to the metaloggers (if present) and to spare
master servers in Pro version.

File data is divided into fragments (chunks) with a maximum size of
64MiB each. Each chunk is itself a file on selected disks on data
servers (chunkservers).

High reliability is achieved by configuring as many different data
servers as appropriate to assure the “” value (number of copies to keep)
set for the given file.

How does the system work
------------------------

| All file operations on a client computer that has mounted MooseFS are
  exactly the same as they would be with other file systems. The
  operating system’s kernel transfers all file operations to the FUSE
  module, which communicates with the process. The process communicates
  through the network subsequently with the managing server and data
  servers (chunk servers). This entire process is fully transparent to
  the user.
| communicates with the managing server every time an operation on file
  metadata is required:

-  creating files

-  deleting files

-  reading directories

-  reading and changing attributes

-  changing file sizes

-  at the start of reading or writing data

-  on any access to special files on

uses a direct connection to the data server (chunk server) that stores
the relevant chunk of a file. When writing a file, after finishing the
write process the managing server receives information from to update a
file’s length and the last modification time.

Furthermore, data servers (chunk servers) communicate with each other to
replicate data in order to achieve the appropriate number of copies of a
file on different machines.

Fault tolerance
---------------

Administrative commands allow the system administrator to specify the
“”, or number of copies that should be maintained, on a per-directory or
per-file level. Setting the goal to more than one and having more than
one data server will provide fault tolerance. When the file data is
stored in many copies (on more than one data server), the system is
resistant to failures or temporary network outages of a single data
server.

| This of course does not refer to files with the “” set to 1, in which
  case the file will only exist on a single data server irrespective of
  how many data servers are deployed in the system.
| Exceptionally important files may have their set to a number higher
  than two, which will allow these files to be resistant to a breakdown
  of more than one server at the same time.
| In general the setting for the number of copies available should be
  one more than the anticipated number of inaccessible or out-of-order
  servers.
| In the case where a single data server experiences a failure or
  disconnection from the network, the files stored within it that had at
  least two copies, will remain accessible from another data server. The
  data that is now ’under its goal’ will be replicated on another
  accessible data server to again provide the required number of copies.
| It should be noted that if the number of available servers is lower
  than the “” set for a given file, the required number of copies cannot
  be preserved. Similarly if there are the same number of servers as the
  currently set goal and if a data server has reached 100% of its
  capacity, it will be unable to hold a copy of a file that is now below
  its goal due to another data server going offline. In these cases a
  new data server should be connected to the system as soon as possible
  in order to maintain the desired number of copies of the file.
| A new data server can be connected to the system at any time. The new
  capacity will immediately become available for use to store new files
  or to hold replicated copies of files from other data servers.
| Administrative utilities exist to query the status of the files within
  the file system to determine if any of the files are currently below
  their goal (set number of copies). This utility can also be used to
  alter the goal setting as required.
| The data fragments stored in the chunks are versioned, so
  re-connecting a data server with older copy of data (i.e. if it had
  been offline for a period of time), will not cause the files to become
  incoherent. The data server will synchronize itself to hold the
  current versions of the chunks, where the obsolete chunks will be
  removed and the free space will be reallocated to hold the new chunks.
| Failures of a client machine (that runs the process) will have no
  influence on the coherence of the file system or on the other clients’
  operations. In the worst case scenario the data that has not yet been
  sent from the failed client computer may be lost.

Platforms
---------

MooseFS is available on every Operating System with a working FUSE
implementation:

-  Linux (Linux 2.6.14 and up have FUSE support included in the official
   kernel)

-  FreeBSD

-  MacOS X

-  OpenIndiana Hipster

The Master Server, Metalogger and Chunkservers can also be run on
Windows with Cygwin. Unfortunately without FUSE it won’t be possible to
mount the filesystem within this operating system.

Moose File System Requirements
==============================

Network requirements
--------------------

MooseFS requires TCP/IP network. The faster the network is, the better
is performance. It is recommended to connect all servers to the same
switch or at least try to minimize network latencies, because they may
have significant impact on performance.

MooseFS requires the following ports to be open (it can be configured in
appropriate configuration files):

-  – Master Server(s)

-  – Chunkservers

-  – CGI Server

Requirements for Master Servers
-------------------------------

As the managing server (master) is a crucial element of MooseFS, it
should be installed on a machine which guarantees high stability and
access requirements which are adequate for the whole system. It is
advisable to use a server with a redundant power supply, ECC memory, and
disk array RAID 1 / RAID 5 / RAID 10. The managing server OS has to be
POSIX compliant (systems verified so far: Linux, FreeBSD, MacOS X and
OpenSolaris).

CPU
~~~

Because Master Server is a single-threaded process, it is recommended to
use modern CPU with high clock (e.g. 3.7 GHz) and small number of cores
(e.g. 4) – especially in MooseFS instances which handle a lot of small
files.

Additionally, disabling CPU power management in BIOS (or enable mode
like “maximum performance”) may have positive impact on efficiency.

You can compare CPUs on the following website – please pay attention to
“single-thread points”: .

RAM size
~~~~~~~~

The most important factor in sizing requirements for the Master Server
machine is RAM, as the full file system structure is cached in RAM for
speed. The Master Server should have approximately 300-350 MiB of RAM
allocated to handle 1 million objects (files, directories, pipes,
sockets, ...).

:

-  Leader Master RAM usage: ( Bytes exactly)

-  “All FS objects” (from MFS CGI):

-  Bytes per one object

HDD free space
~~~~~~~~~~~~~~

The necessary size of HDD depends both on the number of files and chunks
used (main metadata file) and on the number of operations made on the
files (metadata changelog); for example the space of 20 GiB is enough
for storing information for 25 million files and for changelogs to be
kept for up to 50 hours.

You can calculate the minimum amount of space we recommend using the
following formula:

-  – amount of RAM

-  – number of metadata change log files, default is 50 (from )

-  – number of previous metadata files to be kept (default is 1) (also
   from )

| :

(If default values from are used, it is )

The value (before multiplying by ) is an estimation of size used by one
file. On highly loaded MooseFS instance it uses a bit less than 1 GB.

| :
| If you have 128 GiB of RAM, using the formula above, you should
  reserve for :

128\*3 + 51 = 384 + 51 = **435 GiB minimum**.

Requirements for Metalogger(s)
------------------------------

| MooseFS metalogger simply gathers metadata backups from the MooseFS
  Master Server – so the hardware requirements are not higher than for
  the Master Server itself; it needs about the same disk space.
  Similarly to the Master Server – the OS has to be POSIX compliant
  (Linux, FreeBSD, Mac OS X, OpenSolaris, etc.).
| MooseFS Metalogger should have at least the same amount of HDD space
  (**especially the free space in !**) as the main Master Server.

If you would like to use the Metalogger as a Master Server in case of
the main Master’s failure, the Metalogger machine should have at least
the same amount of RAM as the main Master Server.

Requirements for Chunkservers
-----------------------------

Chunkservers, like other MooseFS machines have to have POSIX compliant
OS.

CPU
~~~

MooseFS Chunkserver is a multi-threaded process, so the best choice is
to have a CPU with a number of cores.

RAM size
~~~~~~~~

MooseFS Chunkserver uses approximately 250 MiB of RAM allocated to
handle 1 million chunks.

:

-  Chunkserver RAM usage:

-  Chunks stored on this Chunkserver (from MFS CGI):

-  Bytes per one chunk

HDD space
~~~~~~~~~

Chunkserver machines should have appropriate disk space (dedicated
exclusively for MooseFS). Typical and recommended usage is to create one
partition on each HDD, mount them and enter paths to mounted partitions
in .

Minimal configuration should start from several gigabytes of storage
space (only disks with more than 256 MB and Chunkservers reporting more
than 1 GB of total free space are accessible for new data).

Requirements for Clients / Mounts
---------------------------------

requires FUSE to work; FUSE is available on several operating systems:
Linux, FreeBSD, OpenSolaris and MacOS X, with the following notes:

-  In case of Linux a kernel module with API 7.8 or later is required
   (it can be checked with dmesg command – after loading kernel module
   there should be a line fuse init (API version 7.8)). It is available
   in fuse package 2.6.0 (or later) or in Linux kernel 2.6.20 (or
   later). Due to some minor bugs, the newer module is recommended (fuse
   2.7.2 or Linux 2.6.24, although fuse 2.7.x standalone doesn’t contain
   getattr/write race condition fix).

-  In case of FreeBSD we recommed using fuse-freebsd [2]_, which is a
   successor to fuse4bsd.

-  For MacOSX we recommend using OSXFUSE [3]_, which is a successor to
   MacFUSE and has been tested on MacOSX 10.6, 10.7, 10.8, 10.9 and
   10.11.

Installing MooseFS 3.0
======================

This is a Very Quick Start Guide describing basic MooseFS 3.0
installation in configuration of two Master Servers and three
Chunkservers.

Please note that complete installation process is described in “MooseFS
Step by Step Tutorial”.

For the sake of this document, it’s assumed that your machines have
following IP addresses:

-  Master servers: 192.168.1.1, 192.168.1.2

-  Chunkservers: 192.168.1.101, 192.168.1.102 and 192.168.1.103

-  Users’ computers (clients): 192.168.2.x

In this tutorial it is assumed that you have MooseFS 3.0 Pro version. If
you use MooseFS 3.0 (non-Pro), please remove ’’ from packages names.

In this tutorial it is also assumed that you have Ubuntu/Debian
installed on your machines. If you have another distribution, please use
appropriate package manager instead of .

Note, that most of commands below are preceded by sign, which means,
that you have to run such command as ( sign means normal user). The
easiest way to become is to run:

::

        $ sudo su -
        

Configuring DNS Server
----------------------

| Before you start installing MooseFS, you need to have working DNS.
  It’s needed for MooseFS to work properly with several master servers,
  because DNS can resolve one host name as more than one IP address.
| All IPs of machines which will be master servers must be included in
  DNS configuration file and resolved as “” (or any other selected
  name), e.g.:

::

        mfsmaster   IN  A   192.168.1.1     ; address of first master server
        mfsmaster   IN  A   192.168.1.2     ; address of second master server
            

More information about configuring DNS server is included in supplement
to “MooseFS Step by Step Tutorial”.

Adding repositories
-------------------

Before installing MooseFS you need to add MooseFS Official Supported
Repositories to your system.

Ubuntu / Debian
~~~~~~~~~~~~~~~

First, add the key:

::

        # wget -O - http://ppa.moosefs.com/moosefs.key | apt-key add -
                

Then add the appropriate entry in :

-  | For Ubuntu 14.04 Trusty:

-  | For Ubuntu 12.04 Precise:

-  | For Ubuntu 10.10 Maverick:

-  | For Debian 7.0 Wheezy:

-  | For Debian 6.0 Squeeze:

-  | For Debian 5.0 Lenny:

| After that do:

RedHat / CentOS (EL7)
~~~~~~~~~~~~~~~~~~~~~

| Red Hat 7 familiy OS use Linux system and service manager to start
  processes. To use systemctl command to start MooseFS processes use
  this steps to add repository.
| Add the appropriate key to package manager:

::

        # curl "http://ppa.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
                

Next you need to add the repository entry to yum repo:

::

        # curl "http://ppa.moosefs.com/MooseFS-3-el7.repo" > /etc/yum.repos.d/MooseFS.repo
        # yum update
                

RedHat / CentOS (EL6)
~~~~~~~~~~~~~~~~~~~~~

| Red Hat 6 family OS use runlevel system to start processes. To use
  service command to start MooseFS processes use this steps to add SysV
  repository.
| Add the appropriate key to package manager:

::

        # curl "http://ppa.moosefs.com/RPM-GPG-KEY-MooseFS" > /etc/pki/rpm-gpg/RPM-GPG-KEY-MooseFS
                

Next you need to add the repository entry to yum repo:

::

        # curl "http://ppa.moosefs.com/MooseFS-3-el6.repo" > /etc/yum.repos.d/MooseFS.repo
        # yum update
                

Apple MacOS X
~~~~~~~~~~~~~

It’s possible to run all components of the system on Mac OS X systems,
but most common scenario would be to run the client () that enables Mac
OS X users to access resources available in MooseFS infrastructure.

In case of MacOS X – since there’s no default package manager – we
release files containing only binaries without any startup scripts, that
normally are available in Linux packages.

To install MooseFS on Mac please follow these steps:

-  | download and install FUSE for Mac OS X package from

-  | download and install MooseFS packages from

You should be able to mount MooseFS filesystem in issuing the following
command:

If you’ve exported filesystem with additional options like password
protection, you should include those options in invocation as in
documentation.

Differences in package names between MooseFS Pro and MooseFS
------------------------------------------------------------

The packages in MooseFS 3.0 Pro are named according to following
pattern:

-  
-  
-  
-  
-  
-  
-  
-  
-  

In MooseFS 3.0 (non-Pro) the packages are named according to the
following pattern:

-  
-  
-  
-  
-  
-  
-  
-  

MooseFS Master Server(s) installation
-------------------------------------

Install package by running the following command:

:

::

        # apt-get install moosefs-pro-master
            

:

::

        # yum install moosefs-pro-master
            

Sample configuration files will be created in with the extension
(MooseFS 3.0+) or (MooseFS 2.0). Use these files as your target
configuration files:

::

        # cd /etc/mfs
        # cp mfsmaster.cfg.sample mfsmaster.cfg
        # cp mfsexports.cfg.sample mfsexports.cfg
            

File specifies which users’ computers can mount the file system and with
what privileges. For example, to specify that only machines addressed as
can use the whole structure of MooseFS resources () in read/write mode,
in the first line which is not commented out change the asterisk () to ,
so that you’ll have:

::

        192.168.2.0/24      /       rw,alldirs,maproot=0
            

Now, if you use MooseFS Pro, place proper file into directory. This file
**must** be available on **all** Master Servers.

At this point it is possible to run the MooseFS Master Server:

::

        # mfsmaster start
            

If you use init script manager, which is by default available in Debian,
Ubuntu and RedHat 6 family operating systems, you can also start Master
by issuing the following command:

::

        # service moosefs-pro-master start
            

To start MooseFS Master Server with latest Linux system and service
manager, which is available in RedHat 7 family operating systems, use
this command:

::

        # systemctl start moosefs-pro-master.service
            

You need to repeat these steps on each machine intended for running
MooseFS Master Server (in this example – on and ).

You can also find more detailed description how to add Master Followers
in **MooseFS Upgrade Guide - Chapter 6: Adding master follower(s)
server(s) procedure** (Pro only).

MooseFS CGI Monitor, CGI Server and Command Line Interface installation
-----------------------------------------------------------------------

MooseFS CGI Monitor and MooseFS CGISERV can be installed on any machine,
but good practice tells that it should be installed on every Master
Server.

MooseFS Command Line Interface (CLI) tool allows you to see various
information about MooseFS status. The with option displays basic info
similar to the “Info” tab in CGI. To install CGI, CGISERV and CLI, use
the following commands.

:

::

        # apt-get install moosefs-pro-cgi
        # apt-get install moosefs-pro-cgiserv
        # apt-get install moosefs-pro-cli
            

Set variable to in file to configure autostart.

:

::

        # yum install moosefs-pro-cgi
        # yum install moosefs-pro-cgiserv
        # yum install moosefs-pro-cli
        

Run MooseFS CGI Monitor with :

::

        # service moosefs-pro-cgiserv start
            

Run MooseFS CGI Monitor with :

::

        # systemctl start moosefs-pro-cgiserv.service
            

MooseFS CGI Monitor website should now be available at
http://192.168.1.1:9425 address(for the moment there would be no data
about chunk servers).

Chunk servers installation
--------------------------

:

::

        # apt-get install moosefs-pro-chunkserver
            

:

::

        # yum install moosefs-pro-chunkserver
            

Now you need to prepare basic configuration files for the :

::

        # cd /etc/mfs
        # cp mfschunkserver.cfg.sample mfschunkserver.cfg
        # cp mfshdd.cfg.sample mfshdd.cfg
            

In the file you’ll give locations in which you have mounted hard
drives/partitions designed for the chunks of the system. It is
recommended that they are used exclusively for the MooseFS – this is
necessary to manage the free space properly. For example if you’ll use
and locations, add these two lines to file:

::

        /mnt/mfschunks1
        /mnt/mfschunks2
            

Before you start chunkserver, make sure that the user has rights to
write in the mounted partitions (which is necessary to create a .lock
file):

::

        # chown -R mfs:mfs /mnt/mfschunks1
        # chown -R mfs:mfs /mnt/mfschunks2
            

At this moment you are ready to start the chunk server:

For init script system

::

        # service moosefs-pro-chunkserver start
            

For Linux system and service manager

::

        # systemctl start moosefs-pro-chunkserver.service
            

You need to repeat these steps on each machine intended for running
MooseFS Chunkserver (in this example – on , and .

Now at http://192.168.1.1:9425 full information about the system is
available, including the master server and chunk servers.

MooseFS Clients installation
----------------------------

MooseFS client uses library. During installation process, your operating
system also downloads and installs library if it is not installed.

:

::

        # apt-get install moosefs-pro-client
            

:

::

        # yum install moosefs-pro-client
            

Let’s assume that you want to mount the MooseFS share in a folder on a
client’s machine. Issue the following commands:

::

        # mkdir -p /mnt/mfs
        # mfsmount /mnt/mfs -H mfsmaster
            

Now after running the command you should get information similar to
this:

::

        /storage/mfschunks/mfschunks1
            2.0G    69M     1.9G    4%  /mnt/mfschunks1
        /storage/mfschunks/mfschunks2
            2.0G    69M     1.9G    4%  /mnt/mfschunks2
        mfs#mfsmaster:9421
            3.2G    0       3.2G    0%  /mnt/mfs
            

You need to repeat these steps on each machine intended to be MooseFS
3.0 Client (in this example – on .

To enable MooseFS Client automount during boot, first of all check if
the and packages are installed. If and packages are installed, add
similar entry to the following one in :

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

If MooseFS Client has to be mounted on the same machine that MooseFS
Master Server runs, please put the following entry instead of the one
listed above:

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsdelayedinit,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

Enabling MooseFS services during OS boot
----------------------------------------

Each operating system has it’s own method to manage services start
during boot. Below you can find a few examples of enabling MooseFS
autostart in supported operating systems.

RedHat / Centos (EL6)
~~~~~~~~~~~~~~~~~~~~~

:

To enable MooseFS Chunkserver autostart during OS boot, use command like
in example below:

::

        chkconfig moosefs-chunkserver on
            

:

To enable MooseFS Master Server autostart during OS boot, use command
like in example below:

::

        chkconfig moosefs-master on
            

:

To enable MooseFS Client automount during boot, first of all check if
the and packages are installed:

::

        # rpm -qa | grep fuse
        fuse-2.8.3-4.el6.x86_64
        fuse-libs-2.8.3-4.el6.x86_64
            

If and packages are installed, add similar entry to the following one in
:

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

If MooseFS Client has to be mounted on the same machine that MooseFS
Master Server runs, please put the following entry instead of the one
listed above:

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsdelayedinit,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

RedHat / Centos (EL7)
~~~~~~~~~~~~~~~~~~~~~

In operating systems with , use command to manage init processes at
boot:

:

To enable MooseFS Chunkserver autostart during OS boot:

::

        systemctl enable moosefs-chunkserver.service
            

:

To enable MooseFS Master Server autostart during OS boot:

::

        systemctl enable moosefs-master.service
            

:

To enable MooseFS Client automount during boot, first of all check if
the and packages are installed:

::

        # rpm -qa | grep fuse
        fuse-2.9.2-6.el7.x86_64
        fuse-libs-2.9.2-6.el7.x86_64
            

If and packages are installed, add similar entry to the following one in
:

::

        mfsmount    /mnt/mfs    fuse    mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

If MooseFS Client has to be mounted on the same machine that MooseFS
Master Server runs, please put the following entry instead of the one
listed above:

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsdelayedinit,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

Debian / Ubuntu
~~~~~~~~~~~~~~~

This method works in Debian 6, Debian 7, Ubuntu 12, Ubuntu 14.

:

To enable MooseFS Chunkserver autostart during OS boot, find file and
change variable to :

::

        MFSCHUNKSERVER_ENABLE=true
            

:

To enable MooseFS Master Server autostart during OS boot, edit file and
change variable to :

::

        MFSMASTER_ENABLE=true
            

:

To enable MooseFS Client automount during boot, first of all check if
the and packages are installed. If and packages are installed, add
similar entry to the following one in :

::

        mfsmount    /mnt/mfs    fuse    mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

If MooseFS Client has to be mounted on the same machine that MooseFS
Master Server runs, please put the following entry instead of the one
listed above:

::

        mfsmount    /mnt/mfs    fuse    defaults,mfsdelayedinit,mfsmaster=mfsmaster.example.lan,mfsport=9421    0    0 
            

FreeBSD
~~~~~~~

:

To enable MooseFS Chunkserver autostart during OS boot, add an entry to
:

::

        mfschunkserver_enable="YES"
            

:

To enable MooseFS Chunkserver autostart during OS boot, add entry to :

::

        mfsmaster_enable="YES"
            

:

To enable MooseFS Client automount during boot add the following entry
in to let FreeBSD load module during boot:

::

        fuse_load="YES"
            

And add the entry in :

::

        mfsmount_magic /mnt/mfs moosefs rw,mfsmaster=mfsmaster,mountprog=/usr/local/bin/mfsmount,late 0 0 
            

Basic MooseFS use
-----------------

Create in , in which you store files in one copy (setting ):

::

        mkdir -p /mnt/mfs/folder1
            

and , in which you store files in two copies (setting ):

::

        mkdir -p /mnt/mfs/folder2
            

The number of copies for the folder is set with the command:

::

        # mfssetgoal -r 1 /mnt/mfs/folder1
        /mnt/mfs/folder1:
        inodes with goal changed:               0
        inodes with goal not changed:           1
        inodes with permission denied:          0

        # mfssetgoal -r 2 /mnt/mfs/folder2
        /mnt/mfs/folder2:
        inodes with goal changed:               0
        inodes with goal not changed:           1
        inodes with permission denied:          0
            

Stopping MooseFS
----------------

In order to safely stop the MooseFS cluster you have to perform the
following steps:

-  Unmount the file system on all machines using umount command (in our
   example it would be: )

-  | Stop the Chunk Servers processes:
   | For :
   | For :

-  | Stop the Master Server processes (starting from the FOLLOWER, you
     shuould stop the LEADER Master Server as the last one):
   | For :
   | For :

-  | Stop the Metalogger process:
   | For :
   | For :

Storage Classes
===============

Introduction to Storage Classes functionality in MooseFS 3.0
------------------------------------------------------------

What is a Storage Class?
~~~~~~~~~~~~~~~~~~~~~~~~

Since MooseFS 3.0 goal has been extended to Storage Class. Storage
Classes allow you to specify on which Chunkservers copies of files
should be stored. Storage Classes are defined using label expressions.

To maintain compatibility with standard goal semantics, there are
predefined Storage Classes from 1 to 9 that, unless changed behave like
goals from MooseFS 2.0 or 1.6 (see **Subsection “” of Section
[section:moosefs-storage-class-administration-tool]:** of this manual or
). Goal tools simply work only on these classes.

What are labels?
~~~~~~~~~~~~~~~~

Labels are letters (A-Z – 26 letters) that can be assigned to
Chunkservers. Each chunkserver can have multiple (up to 26) labels.

Labels expression is a set of subexpressions separated by commas, each
subexpression specifies the storage schema of one copy of a file.
Subexpression can be: an asterisk or a label schema.

Label schema can be one label or an expression with sums,
multiplications and brackets. Sum means a file can be stored on any
chunkserver matching any element of the sum (logical or). Multiplication
means a file can be stored only on a chunkserver matching all elements
(logical and). Asterisk means any chunkserver.

Identical subexpressions can be shortened by adding a number in front of
one instead of repeating it a number of times.

For more information about labels expressions, refer to **Subsection “”
of Section [section:moosefs-storage-class-administration-tool]:** of
this manual.

How to use Storage Classes?
---------------------------

Machines configuration
~~~~~~~~~~~~~~~~~~~~~~

In this example we have MooseFS 3.0 installed on 11 machines:

-  , – Master Servers

-  – Chunkservers

Assumption:

-  On the MooseFS instance there is some initial data stored with goal
   (Storage Class ).

Example of MooseFS installation without Storage Classes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To run MooseFS without any user-defined Storage Classes, you don’t have
to make any changes in configuration. Just install MooseFS with default
configuration. The process is described in “**MooseFS Step by Step
Tutorial**”.

The picture below shows the discussed installation:

|image|

If labels on Chunkservers are not set up, the system is balanced like
MooseFS 2.0. The image below presents system balance at this point:

|image|

Labelling Chunkservers
~~~~~~~~~~~~~~~~~~~~~~

To add labels to the system, i.e. assign them to Chunkservers, you need
to edit their configuration files (). Open the file, uncomment the
following line and after the equation character type labels you want to
set on specific Chunkserver. For example to set label on Chunkservers
ts04, ts05, ts06 and ts07, their configuration should look like this:

::

        [...]
        
        # labels string (default is empty - no labels)
        LABELS = A
        
        [...]
                

The next step is to “inform” the Chunkserver, that the Configuration
file has changed. Issue the command:

::

        root@chunkserver:~# service moosefs-pro-chunkserver reload
                

or:

::

        root@chunkserver:~# mfschunkserver reload
                

Similarly set label for Chunkservers ts08, ts09, ts10, ts11, ts12.

After this step in CGI monitor you can observe, that Chunkservers
ts04..ts07 have label and Chunkservers ts08..ts12 – label :

|image|

: If you want to set more than one label for a Chunkserver, just enter
appropriate labels in configuration file (). MooseFS supports schemes
listed below, so you can choose the one, which fits for you the best,
e.g.:

::

        [...]
        # labels string (default is empty - no labels)
        LABELS = XYZ
        [...]
                

or:

::

        [...]
        # labels string (default is empty - no labels)
        LABELS = X, Y, Z
        [...]
                

or:

::

        [...]
        # labels string (default is empty - no labels)
        LABELS = X Y Z
        [...]
                

The picture below presents current system configuration:

|image|

Creating Storage Classes
~~~~~~~~~~~~~~~~~~~~~~~~

In order to create a Storage Class on MooseFS, use the tool. Below you
can find a simple example, you can read a full description of usage in
**Chapter [chapter:storage-classes-tools]:** or in .

Let’s create a storage class named :

First of all, mount MooseFS:

::

        root@client:~# mount -t moosefs mfsmaster.test.lan: /mnt/mfs
        mfsmaster 192.168.1.2 - found leader: 192.168.1.3
        mfsmaster accepted connection with parameters: read-write,restricted_ip,admin ; root mapped to root:root
        root@client:~#
                

or

::

        root@client:~# mfsmount -H mfsmaster.test.lan /mnt/mfs
        mfsmaster 192.168.1.2 - found leader: 192.168.1.3
        mfsmaster accepted connection with parameters: read-write,restricted_ip,admin ; root mapped to root:root
        root@client:~#
                

Then, navigate to mounted file system:

::

        root@client:~# cd /mnt/mfs
        root@client:/mnt/mfs#
                

Let’s assume, you want to have your files stored in 2 copies on
Chunkservers labelled as . Create a Storage Class with appropriate
definition:

::

        root@client:/mnt/mfs# mfsscadmin create 2A sclass1
        create ; 0
        storage class make sclass1: ok
        root@client:/mnt/mfs#
                

It means that every file with assigned will be stored in two copies: one
will be kept on Chunkserver with label A, another one – on another
Chunkserver with label A.

Similarly, create a Storage Class , which keep 2 copies on Chunkservers
labelled as :

::

        root@client:/mnt/mfs# mfsscadmin create 2B sclass2
        create ; 0
        storage class make sclass2: ok
        root@client:/mnt/mfs#
                

: You don’t have to navigate to mounted file system to create a Storage
Class – it is also possible to do it from any location. In such case
just let tool know, where MooseFS is mounted (in first parameter), e.g.:

::

        root@client:~# mfsscadmin /mnt/mfs create 2B sclass2
                

It applies to all Storage Classes tools.

Listing Storage Classes
~~~~~~~~~~~~~~~~~~~~~~~

Now, let’s check, if the classes has been properly created and are
available to use:

::

        root@client:/mnt/mfs# mfsscadmin list
        list ; 1
        1
        2
        3
        4
        5
        6
        7
        8
        9
        sclass1
        sclass2
        root@client:/mnt/mfs#
                

You can also see more detailed view by issuing the command with switch:

::

        root@client:/mnt/mfs# mfsscadmin list -l
        list ; 1
        [...]
        sclass1 : 2 ; admin_only: NO ; create_mode: STD ; create_labels: [A] , [A] ; keep_labels: [A] , [A]
        sclass2 : 2 ; admin_only: NO ; create_mode: STD ; create_labels: [B] , [B] ; keep_labels: [B] , [B]
        root@client:/mnt/mfs#
                

Assigning Storage Class to files / directories
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are several tools to manage Storage Classes assignment to files,
directories etc.: , , , , . You can find out more about them in
**Section [section:moosefs-storage-class-management-tools]:** or by
issuing .

Now it’s time to store some data on this MooseFS instance. Create two
directories, let’s say and .

::

        root@client:~# cd /mnt/mfs
        root@client:/mnt/mfs# mkdir dataX
        root@client:/mnt/mfs# mkdir dataY
        root@client:/mnt/mfs#
                

Next, assign Storage class to :

::

        root@client:/mnt/mfs# mfssetsclass sclass1 dataX
        dataX: storage class: 'sclass1'
        root@client:/mnt/mfs#
                

It means that this directory, its subdirectories, files and so on will
be stored according to policy.

Similarly, assign Storage class to :

::

        root@client:/mnt/mfs# mfssetsclass sclass2 dataY
        dataY: storage class: 'sclass2'
        root@client:/mnt/mfs#
                

It means that this directory, its subdirectories, files and so on will
be stored according to policy.

For more information about assigning Storage Classes to files, refer to
**Section [section:moosefs-storage-class-management-tools]:** .

Now on MooseFS Monitor (“Resources” tab) you can observe, that goal is
set and it can be fulfilled.

|image|

Creating files
^^^^^^^^^^^^^^

In this step you will create some files in previously created
directories ( and ) to fill MooseFS instance with data. This operation
may take some time. Issue the following commands:

::

        root@client:/mnt/mfs# cd dataX
        root@client:/mnt/mfs/dataX# for i in `seq 1 35`; do dd if=/dev/urandom of=dd1G_$i.bin bs=1M count=1024; done
        [...]
        root@client:/mnt/mfs/dataX# cd ../dataY
        root@client:/mnt/mfs/dataY# for i in `seq 1 10`; do dd if=/dev/urandom of=dd1G_$i.bin bs=1M count=1024; done
        [...]
        root@client:/mnt/mfs/dataY#
                    

: These commands create approx. 90 GiB (45 GiB multiplied by goal 2) of
data – 35 GiB in directory (RAW size: 70 GiB) and 10 GiB in directory
(RAW size: 20 GiB), so adjust them for your testing purposes.

Filesystem balance with Storage Classes applied
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now you can observe, that filesystem is balanced according to Storage
Classes policy: Chunkservers with label A store the data data with goal
applied, similarly – Chunkservers with label B store the data with goal
:

|image|

**Notice, that the system looks “unbalanced”, but it is, in fact,
balanced as much, as the requirements of Storage Classes allow it to
be.**

Also in tab “Resources” number of inodes has changed:

|image|

Creation, keep, archive labels
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In MooseFS 3.0 a possibility to “plan” changing labels has been added.

Now you can “tell” MooseFS (crate appropriate Storage Class), what label
expression it should use for file(s) while creating it (them), to what
label expression change it after the creation and to what label
expression change it after a specific time since last modification.

You can define it while creating a Storage Class by tool.

Synopsis
^^^^^^^^

| 

Creation labels
^^^^^^^^^^^^^^^

“Creation labels” () – optional parameter, that tells the system to
which Chunkservers, defined by the expression, the chunk should be first
written just after creation; if this parameter is not provided for a
class, the Chunkservers will be used.

Keep labels
^^^^^^^^^^^

“Keep labels” () – mandatory parameter (assumed in the second,
abbreviated version of the command), that tells the system on which
Chunkservers, defined by the expression, the chunk(s) should be kept
always, except for special conditions like creating and archiving, if
defined.

Archive labels
^^^^^^^^^^^^^^

“Archive labels” () – optional parameter, that tells the system on which
Chunkservers, defined by the expression, the chunk(s) should be kept for
archiving purposes; the system starts to treat a chunk as archive, when
the last () of the file it belongs to is older than the number of days
specified with parameter.

How to set it?
^^^^^^^^^^^^^^

For more information about the command to issue, refer to **Section
[section:moosefs-storage-class-administration-tool]:** or issue .

Chunkserver states
~~~~~~~~~~~~~~~~~~

Chunkserver can work in 3 states: , and (since MooseFS 3.0.62) :

-  state is a standard state. In “Servers” CGI tab you can see load as a
   normal number, e.g.: .

-  state is a special Chunkserver state. It is activated when e.g. you
   add a new, empty HDD to a Chunkserver. Then Chunkserver enters this
   special mode and rebalances chunks between all HDDs to make all HDDs
   utilization as close to equal as possible. In “Servers” CGI tab you
   can see load as number in round brackets, e.g.: .

-  is a special, **temporary** Chunkserver state. It is activated when
   Chunkserver load is high and Chunkserver is not able to perform more
   operations at the moment. In such case, Chunkserver sends an
   information to Master Server that it is overloaded. If the load
   lowers to the normal level, Chunkserver sends an information to
   Master Server, that it is not overloaded any more. In “Servers” CGI
   tab you can see load as a number in square brackets, e.g.: .

Chunk creation modes
~~~~~~~~~~~~~~~~~~~~

While you store your data on labelled Chunkservers, a situation may
occur that there is no more space on appropriate Chunkservers or they
are overloaded.

To decide what MooseFS should do when free space ends or when
Chunkserver you want to store data to is overloaded, you need to use
creating chunks modes.

You can define these modes for each file, directory, it’s subdirectories
and so on, because they can be set (or modified) when you set the goal
for your data.

There are three modes:

-  mode ( flag to ) – in this mode the system will use other servers in
   case of overloaded servers or no space on servers and will replicate
   data to correct servers when it becomes possible.

-  mode (no flag or flag to ) – in case of overloaded servers system
   will wait for them, but in case of no space available will use other
   servers and will replicate data to correct servers when it becomes
   possible.

-  mode ( flag to ) – in this mode the system will return error () in
   case of no space available on servers marked with labels specified
   for chunk creation. It will still wait for overloaded servers.

A table below presents MooseFS behavior for these modes:

+---------------+---------------------------------+----------------------------------+
|               | **Chunkserver is full**         | **Chunkserver is overloaded**    |
+===============+=================================+==================================+
| **Loose**     | use servers with other labels   | use servers with other labels    |
+---------------+---------------------------------+----------------------------------+
| **Default**   | use servers with other labels   | wait for available Chunkserver   |
+---------------+---------------------------------+----------------------------------+
| **Strict**    | no write (returns )             | wait for available Chunkserver   |
+---------------+---------------------------------+----------------------------------+

You can observe current states in Resources CGI tab.

Preferred labels during read/write (in )
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to specify preferred labels for choosing Chunkservers
during read and write operations at the MooseFS Client () side:

::

                -o mfspreflabels=LABELEXPR
                      specify preferred labels for choosing Chunkservers during I/O
                

You can set different preferred labels for each mountpoint.

Preferred labels in MooseFS Client are a list (up to 9) of labels
expressions, e.g. :math:`E_{1}`, :math:`E_{2}`, :math:`E_{3}`.

While a client performs a read operation, Master Server returns a list
of chunks’ locations (in random order) in the following form (CS means
Chunkserver): :math:`CS_{a}`, :math:`CS_{b}`, :math:`CS_{c}`, ...

Each of :math:`CS_{x}` entry contains a list of labels assigned to
specific Chunkserver.

Priority of each :math:`CS_{x}` is calculated as the minimum :math:`y`
value, where labels from :math:`CS_{x}` match expression :math:`E_{y}`.
If no expression matches, the priority is set as a number of expressions
:math:`+1`.

The lowest number means the highest priority.

Then, the list of Chunkservers is sorted by priorities. The first
Chunkserver from the list (which has the highest priority / the lowest
number) is used while reading.

If more than one Chunkserver has the same priority, Client picks the one
that got the least number of operations from this Client so far.

If a specific chunk read ends with an error, Client can use a chunk copy
with lower priority (greater number).

In case of writing, the list of Chunkservers is sorted similarly and
data is written to Chunkserver with the highest priority. The difference
is, if more that one Chunkserver has the same priority, the order form
Master Server is used.

If no is set, the order of list from MooseFS Master is used with no
further modifications.

Storage Classes tools
---------------------

MooseFS Storage Class administration tool – 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Synopsis
^^^^^^^^

-  
-  
-  
-  
-  
-  
-  

Description
^^^^^^^^^^^

is a tool for defining storage classes, which can be later applied to
MooseFS objects with mfssetsclass, mfsgetsclass etc.

Storage class is a set of labels expressions and options that indicate,
on which chunkservers the files in this class should be written and
later kept.

Commands
^^^^^^^^

-  creates a new storage class with given options, described below and
   names it ; there can be more than one name provided, multiple storage
   classes with the same definition will be created then

-  – changes the given options in a class or classes indicated by
   paremeter(s)

-  – removes the class or classes indicated by paremeter(s); if any of
   the classes is not empty (i.e. it is still used by some MooseFS
   objects), it will not be removed and the tool will return an error
   and an error message will be printed; empty classes will be removed
   in any case

-  – copies class indicated by under a new name provided with

-  – changes the name of a class from to

-  – lists all the classes

Options
^^^^^^^

-  – optional parameter, that tells the system to which chunkservers,
   defined by the expression, the chunk should be first written just
   after creation; if this parameter is not provided for a class, the
   chunkservers will be used

-  – mandatory parameter (assumed in the second, abbreviated version of
   the command), that tells the system on which chunkservers, defined by
   the expression, the chunk(s) should be kept always, except for
   special conditions like creating and archiving, if defined

-  – optional parameter, that tells the system on which chunkservers,
   defined by the expression, the chunk(s) should be kept for archiving
   purposes; the system starts to treat a chunk as archive, when the
   last modification time of the file it belongs to is older than the
   number of days specified with option

-  – optional parameter that **must** be defined when is defined,
   parameter defines after how many days from last modification time a
   file (and its chunks) are treated as archive

-  – can be either 1 or 0 and indicates if the storage class is
   available to everyone (0) or admin only (1)

-  – force the changes on a predefined storage class (see below), use
   with caution!

-  – is described below in “Creation modes” section

-  – list also definitions, not only the names of existing storage
   classes

Labels expressions
^^^^^^^^^^^^^^^^^^

Labels are letters (A-Z – 26 letters) that can be assigned to
chunkservers. Each chunkserver can have multiple (up to 26) labels.
Labels are defined in file, for more information refer to the
appropriate manpage.

Labels expression is a set of subexpressions separated by commas, each
subexpression specifies the storage schema of one copy of a file.
Subexpression can be: an asterisk or a label schema.

Label schema can be one label or an expression with sums,
multiplications and brackets. Sum means a file can be stored on any
chunkserver matching any element of the sum (logical or).

Multiplication means a file can be stored only on a chunkserver matching
all elements (logical and). Asterisk means any chunkserver. Identical
subexpressions can be shortened by adding a number in front of one
instead of repeating it a number of times.

:

-  – files will have two copies, one copy will be stored on
   chunkserver(s) with label , the other on chunkserver(s) with label

-  – files will have two copies, one copy will be stored on
   chunkserver(s) with label , the other on any chunkserver(s)

-  – files will have two copies, stored on any chunkservers (different
   for each copy)

-  – files will have two copies, one copy will be stored on any
   chunkserver(s) that has both labels and (**multiplication of
   labels**), the other on any chunkserver(s) that has either the label
   or the label (**sum of labels**)

-  – files will have three copies, one copy will be stored on any
   chunkserver(s) with A label, the second on any chunkserver(s) that
   has the label and either or label, the third on any chunkserver(s),
   that has the label and either or label

-  expression is equivalent to expression

-  expression is equivalent to expression

-  expression is equivalent to expression is equivalent to expression

Creation modes
^^^^^^^^^^^^^^

It is important to specify what to do in case when there is no space
available on all servers marked with labels needed for new chunk
creation. Also all servers marked with such labels can be temporarily
overloaded. The question is if the system should create chunks on other
servers or not.

Answer to this question should be resolved by user and hence the option.

-  By default (no options or option ) in case of overloaded servers
   system will wait for them, but in case of no space available will use
   other servers and will replicate data to correct servers when it
   becomes possible.

-  Option turns on mode. In this mode the system will return error () in
   case of no space available on servers marked with labels specified
   for chunk creation. It will still wait for overloaded servers.

-  Option turns on mode. In this mode the system will use other servers
   in case of overloaded servers or no space on servers and will
   replicate data to correct servers when it becomes possible.

Predefined Storage Classes
^^^^^^^^^^^^^^^^^^^^^^^^^^

For compatibility reasons, every fresh or freshly upgraded instance of
MooseFS has 9 predefined storage classes. Their names are single digits,
from to , and their definitions are to .

They are equivalents of simple numeric goals from previous versions of
the system. In case of an upgrade, all files that had goal before
upgrade, will now have storage class.

These classes can be modified only when option is specified. It is
advised to create new storage classes in an upgraded system and migrate
files with mfsxchgsclass tool, rather than modify the predefined
classes. The predefined classes **cannot** be deleted nor renamed.

MooseFS Storage Class management tools – 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Synopsis
^^^^^^^^

-  
-  
-  
-  
-  

Description
^^^^^^^^^^^

These tools operate on object’s Storage Class name. This is an extended
version of classic goal. There are predefined storage classes provided
as equivalents of goals 1 to 9 (names are simply , , ... , ). Other
classes can be created / modified / deleted etc. by administrator using
tool.

-  prints current storage class of given object(s). option enables
   recursive mode, which works as usual for every given file, but for
   every given directory additionally prints current storage class of
   all contained objects (files and directories).

-  changes current storage class of given object(s). option enables
   recursive mode.

-  copies storage class from one object to given object(s).

-  sets storage class to of given objects(s) but only when current
   storage class is set to .

-  lists currently defined storage classes. option enables long format –
   whole class definition is printed for each class, not only its name.
   For description of storage class definition refer to mfsscadmin
   manpage.

General options
^^^^^^^^^^^^^^^

Most of mfstools use , , , , and options to select format of printed
numbers.

-  causes to print exact numbers,

-  uses binary prefixes (Ki, Mi, Gi as :math:`2^{10}`, :math:`2^{20}`
   etc.) while uses SI prefixes (k, M, G as :math:`10^{3}`,
   :math:`10^{6}` etc.).

-  , and - show plain numbers respectivaly in kibis (binary kilo –
   1024), mebis (binary mega – :math:`1024^{2}`) and gibis (binary giga
   – :math:`1024^{3}`).

The same can be achieved by setting environment variable to: 0 (exact
numbers), 1 or h (binary prefixes), 2 or H (SI prefixes), 3 or h+ (exact
numbers and binary prefixes), 4 or H+ (exact numbers and SI prefixes).
The default is to print just exact numbers.

Inheritance
^^^^^^^^^^^

When new object is created in MooseFS, attributes such as storage class,
trashtime and extra attributes are inherited from parent directory. So
if you set i.e. “noowner” attribute and storage class to “important” in
a directory then every new object created in this directory will have
storage class set to “important” and “noowner” flag set.

A newly created object inherits always the current set of its parent’s
attributes. Changing a directory attribute does not affect its already
created children. To change an attribute for a directory and all of its
children use option.

Common use scenarios
--------------------

Scenario 1: Two server rooms (A and B)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let’s assume that chunkservers with label A are in server room A, and
with label B – in server room B (divided exactly as in steps above):

|image|

Using Storage Classes, you can simply decide, which server room your
data is stored to.

: Slow link between the sites (server room A and server room B in above
example) will slow down I/O write operations to files with chunks stored
in both sites due to synchronous nature of I/O write operations. Because
of that reason alone, it is recommended to have a very fast connection
between sites.

Scenario 2: SSD and HDD drives
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let’s assume, that chunkservers ts04..ts07 have SSD drives and
chunkservers ts08..ts12 have HDD drives. For example, you can label
chunkservers with HDD drives as , and with SSD drives – as :

|image|

You can configure Storage Classes, so that your frequently used data is
stored on SSD Chunkservers (e.g. Storage Class ), and data not accessed
very often – on HDD Chunkservers (e.g. Storage Class ).

You can also easily move some data (e.g. after end of the year) from SSD
to HDD chunkservers – you just need to change the Storage Class
assignment from to for this data and MooseFS will automatically take
care of moving process.

: you have a directory named located on MooseFS mountpoint. This
directory and its subdirectories and files are used very often by a lot
of processes. You want to:

-  store this directory in four copies – these are very important files

-  speed up access to this directory,

so you set up and define a Storage Class e.g. defined as (four copies on
Chunkservers with fast, SSD drives) and assign it to the directory
recursively. Issue the commands below:

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create 4S 4ssdcopies
                create ; 0
                storage class make 4ssdcopies: ok
                
                root@client:/mnt/mfs# mfssetsclass -r 4ssdcopies Reports2015
                Reports2015:
                 inodes with storage class changed:              5685
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

But year 2015 has passed, and now is used infrequently and you want to
free some space on SSD drives to store new data. So you want to move
this directory, its subdirectories and files to HDD drives and store it
only in three copies.

You just need to set up and define a Storage Class e.g. defined as
(three copies on Chunkservers with HDD drives) and exchange the Storage
Class for files which currently have Storage Class applied with Storage
Class:

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create 3H 3hddcopies
                create ; 0
                storage class make 3hddcopies: ok
                
                root@client:/mnt/mfs# mfsxchgsclass -r 4ssdcopies 3hddcopies Reports2015
                Reports2015:
                 inodes with storage class changed:              5685
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

MooseFS takes care of moving process and your data is safe and
accessible during moving from SSD to HDD drives (Chunkservers).

Scenario 3: Two server rooms (A and B) + SSD and HDD drives
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

|image|

As shown in the picture above, this Scenario is a combination of
Scenario 1 and Scenario 2. Let’s assume, that in two server rooms you
have two types of chunkservers: some of them containing HDD drives, some
– SSD drives.

Now you want to store e.g. frequently used data on chunkservers with SSD
drives and data used from time to time – on chunkservers with HDD
drives. You also want to have a copy of all data in each server room.

In scenario presented above, you need to set the following labels:

-  Server room A, SSD chunkservers: labels and ,

-  Server room A, HDD chunkservers: labels and ,

-  Server room B, SSD chunkservers: labels and ,

-  Server room B, HDD chunkservers: labels and .

Then you need to set up and define appropriate Storage Classes and apply
them to your files.

Directory used very often named – you want to store it in 2 copies on
**SSD** drives (Chunkservers): one copy in server room , another in
server room .

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create AS,BS frequent
                create ; 0
                storage class make frequent: ok
                
                root@client:/mnt/mfs# mfssetsclass -r frequent Frequent
                Frequent:
                 inodes with storage class changed:              564513
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

Directory used from time to time named – you want to store it in 2
copies on **HDD** drives (Chunkservers): one copy in server room ,
another in server room .

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create AH,BH rare
                create ; 0
                storage class make rare: ok
                
                root@client:/mnt/mfs# mfssetsclass -r rare Rare
                Rare:
                 inodes with storage class changed:              497251
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

So your directory (and its subdirectories and files) is stored now on
Chunkservers which have both and labels and on Chunkservers having both
and labels.

Your directory (and its subdirectories and files) is stored now on
Chunkservers which have both and labels and on Chunkservers having both
and labels.

You also want to store your directory named in three copies. You want to
store one copy in server room on SSD chunkservers, and two copies in
server room , either on HDD or SSD chunkservers. Issue the following
commands:

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create AS,2B[H+S] backup
                create ; 0
                storage class make backup: ok
                
                root@client:/mnt/mfs# mfssetsclass -r backup Backup
                Backup:
                 inodes with storage class changed:              879784
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

The labels expression is a *multiplication* and *sum* of labels. For
more information, refer to **Section [section:labels-expressions]:** of
this document.

For more information about and , refer to **Chapter
[chapter:storage-classes-tools]:** of this document.

: Slow link between the sites (server room A and server room B in above
example) will slow down I/O write operations to files with chunks stored
in both sites due to synchronous nature of I/O write operations. Because
of that reason alone, it is recommended to have a very fast connection
between sites.

Scenario 4: Creation, Keep and Archive modes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let’s assume you want to write fast a big amount of important data and
your computer is located closer to server room A than to server room B.
So you want to create chunks in server room A, on SSD chunkservers, in
two copies ().

But your goal is to have one copy of this data in server room A, and the
other one in server room B, both on SSD chunkservers. MooseFS will take
care of the replication process ().

And finally, after 30 days, you want MooseFS to move this data to HDD
chunkservers in both server room A and B ().

First of all, create a directory:

::

                root@client:~# cd /mnt/mfs
                root@client:/mnt/mfs# mkdir ImportantFiles
                

Then, set up and define a Storage Class, e.g. , defined as and assign it
to the newly created directory directory:

::

                root@client:~# cd /mnt/mfs
                
                root@client:/mnt/mfs# mfsscadmin create -C 2AS -K AS,BS -A AH,BH -d 30 important
                create ; 0
                storage class make important: ok
                
                root@client:/mnt/mfs# mfssetsclass important ImportantFiles
                ImportantFiles:
                 inodes with storage class changed:              1
                 inodes with storage class not changed:          0
                 inodes with permission denied:                  0
                 
                root@client:/mnt/mfs#
                

And that’s all! Now you can write the data to this directory.

Your data will be safe, stored very fast on SSD chunkservers in server
room A while creating (you are close to this server room), copied by
MooseFS also to server room B and after 30 days – automatically moved to
HDD chunkservers.

Troubleshooting
===============

Metadata save
-------------

Sometimes MFS master server freezes during the metadata save. To
overcome this problem you should change one setting in your system. On
your master machines, you should enable overcommit memory setting by
issuing the following command as root:

::

        # echo 1 > /proc/sys/vm/overcommit_memory
            

To do it permanently, you can add the following line to your file (it
works only on Linux):

::

        vm.overcommit_memory=1
            

| More detail about the reasons for this behavior:
| Master server performs a fork operation, effectively spawning another
  process to save metadata to disk. Theoretically, when you fork a
  process, the process memory is copied. In real life it is done the
  lazy way – the memory is marked, so that if any changes are to occur,
  a block with changes is copied as needed, but only then. Now, if you
  fork a process that has 180GB of memory in use, the system can “just
  do it”, or check if it has 180GB of free memory and reserve it for the
  forked “child”, and only then do it and, when it doesn’t have enough
  memory, the fork operation fails – this is the case in Linux, so
  actually saving metadata is done in the main process, because fork
  operation failed.

This behavior differs between systems and even between distributions of
one system.

It is safe to enable overcommit memory (the “just do it” way) with
mfsmaster, because the forked process is short lived. It terminates as
soon as it manages to save metadata, and during the time that it works,
there are usually not that many changes to the main process’ memory, so
the amount of additional RAM needed is relatively small.

Alternatively, you can add huge (at least equal to the amount of
physical RAM or even more) amounts of swap space on your master servers
– then the fork should succeed, because it should always find the needed
memory space in your swap.

Master metadata restore from Metaloggers
----------------------------------------

MooseFS (non-Pro) have only one Master Server, but can have several
Metaloggers deployed for backup. If for some reason you loose all
metadata files and changelogs from master server you can use data form
metalogger to restore your data. To start dealing with recovery first
you need to transfer all data stored on metalogger in to master metadata
folder. Files on metalogger will have prefix prepended to the filenames.
After all files are copied, you need to create file from changelogs and
files. To do this we need to use the command . starts to build new
metadata file and starts process.

Maintenance mode
----------------

Maintenance mode in general is helpful when there is need for
maintenance on Chunkserver(s), like Chunkserver package upgrade to a
newer version, adding new HDD / replacing broken ones or system upgrade
(and e.g. reboot).

Maintenance mode has been introduced, because in MooseFS 1.6, when there
was need for maintenance on Chunkserver(s) and necessity to turn
server(s) off, a lot of replications were being performed, because
MooseFS had started to replicate all undergoal chunks from another
available copy to fulfill the goal (it’s one of MooseFS principals).
Then, when it was back again – a lot of deletions were running, because
of presence of overgoal chunks, created during replications. So a lot of
unnecessary I/O operations.

By enabling maintenance mode before stopping Chunkserver(s) process(es)
/ turning machine(s) off or *post factum*, you can prevent MooseFS from
replicating chunks from such turned off Chunkserver(s). **Note:
Server(s) in maintenance mode must match currently off (disconnected)
servers. If they don’t match, all chunks are replicated.**

Additionally, MooseFS treats Chunkservers in maintenance mode as
overloaded (no chunk creations, replications etc.). It means, that new
chunks are not created on Chunkservers in maintenance mode. The reason
of such behavior is because when you want to turn Chunkserver off / stop
the Chunkserver process, at the moment of stopping, some I/O operations
may go to this Chunkserver and when you just stop it, some write
operations must be re-tried (because they haven’t been finished on this
stopped Chunkserver). When you turn maintenance mode on for specific
Chunkserver a few seconds before stop, MooseFS will finish write
operations and won’t start a new ones on this Chunkserver.

**Maintenance mode is designed to be a temporary state and it is not
recommended to put Chunkservers in this mode for a long time.**

You can enable or disable maintenance mode in CGI monitor by clicking
“switch on / switch off” in “maintenance” column, or sending a command
using:

-  – to switch maintenance mode on

-  – to switch maintenance mode off

**Note: If number of Chunkservers in maintenance mode is equal or
greater than 20% of all Chunkserver, MooseFS treats all Chunkservers
like maintenance mode wouldn’t be enabled at all.**

Chunk replication priorities
----------------------------

In MooseFS 2.0 a few chunk replication classes and priorities have been
introduced:

-  Replication limit class 0 and class 1 – replication for data safety

-  Replication limit class 2 and class 3 – equalization of used disk
   space

These classes and priorities are described below:

-  Replication limit class 0 (for endangered chunks):

   -  priority 0: 0 (chunk) copies on regular disks and 1 copy on disk
      marked for removal

   -  priority 1: 1 copy on regular disks and 0 copies on disks marked
      for removal

-  Replication limit class 1 (for undergoal chunks):

   -  priority 2: 1 copy on regular disk and some copies on disks marked
      for removal

   -  priority 3: :math:`>`\ 1 copy on regular disks and at least 1 copy
      on disks marked for removal

   -  priority 4: just undergoal chunks (“goal” :math:`>` “valid
      copies”, no copies on disks marked for removal)

-  Replication limit class 2: Rebalancing between chunkservers with disk
   space usage around arithmetic mean

-  Replication limit class 3: Rebalancing between chunkserver with disk
   space usage strongly above or strongly below arithmetic mean (very
   low or very high disk space usage, e.g. when new chunkserver is
   added)

MooseFS Tools
=============

For MooseFS Master Server(s)
----------------------------

– start, restart or stop Moose File System master process

**SYNOPSIS**

-  
-  
-  

**DESCRIPTION**

is the master program of Moose File System.

**OPTIONS**

-  print version information and exit

-  print usage information and exit

-  specify alternative path of configuration file (default is in system
   configuration directory)

-  log undefined configuration values (when default is assumed)

-  run in foreground, don’t daemonize

-  how long to wait for lockfile (in seconds; default is 1800 seconds)

-  ignore some metadata structure errors

-  automatically restore metadata from change logs

-  start without metadata (usable only in pro version – used to start
   additional masters)

-  produce more verbose output

-  even more verbose output

-  is the one of , , , , or . Default action is restart. The test action
   will yield one of two responses: “” or “”. The kill action will send
   a SIGKILL to the currently running master process. SIGHUP or reload
   action forces to reload all configuration files.

**FILES**

-  configuration file for MooseFS master process; refer to
   mfsmaster.cfg(5) manual for details

-  MooseFS access control file; refer to mfsexports.cfg(5) manual for
   details

-  Network topology definitions; refer to mfstopology.cfg(5) manual for
   details

-  lock file of running MooseFS master process (created in data
   directory)

-  , MooseFS filesystem metadata image (created in data directory)

-  MooseFS filesystem metadata change logs (created in data directory;
   merged into metadata.mfs once per hour)

-  MooseFS master charts state (created in data directory)

– doesn’t exist this version of MooseFS

**DESCRIPTION**

This tool was removed as of version 1.7. To achieve the same effect,
simply start your mfsmaster with parameter.

– dump MooseFS metadata info in human readable format.

**SYNOPSIS**

**DESCRIPTION** dumps MooseFS metadata info in human readable format.
Output consists of several sections with different types of information.
Every section consist of header data – rows starting with hash (#) sign
- and content data (may be empty).

**FILE HEADER**

-  configuration file for MooseFS master process; refer to
   mfsmaster.cfg(5) manual for details

-  – MooseFS version

-  – metadata file version

-  – metadata file id

**SECTION HEADER**

-  – section header (section type + version)

-  – length of section

-  – name of section

-  – hexadecimal representation of section version

**SESS SECTION**

-  – first free session id

-  – number of stats remembered in each session

-  – line describing a single session

   -  – session id

   -  – IP address

   -  – root inode number

   -  – session flags

   -  – mingoal and maxgoal

   -  – mintrashtime and maxtrashtime

   -  – maproot uid,gid and mapall uid,gid

   -  – disconnection time (optional)

   -  – current hour stats data

   -  – last hour stats data

   -  – session name (usually local mount point)

**NODES SECTION**

-  – maximum inode number used by system

-  – number of inodes in hash table

-  – line with node (inode) description

   -  – node type (-,D,S,F,B,C,L,T,R)

      -  – file

      -  – directory

      -  – socket

      -  – fifo

      -  – block device

      -  – character device

      -  – symbolic link

      -  – trash file

      -  – sustained file (removed open file)

   -  – inode number

   -  – labelset number (10+) or goal (1-9)

   -  – flags

   -  – mode

   -  – uid

   -  – gid

   -  – atime, mtime and ctime timestamps

   -  – trashtime

   -  – rdevhi,rdevlo (only block and character devices)

   -  – path (only symbolic links)

   -  – file length (only files)

   -  – chunk list (only files)

   -  – sessions that have this file open (only files)

**EDGES SECTION**

-  – next available edge id (descending)

-  – line with edge description

   -  – parent inode number

   -  – child inode number

   -  – edge id

   -  – edge name

**FREE SECTION**

-  – number of free (reusable) nodes

-  – line with free inode description

   -  – inode number

   -  – deletion timestamp

**QUOTA SECTION**

-  – number of nodes with quota

-  – line with quota description

   -  – inode number

   -  – grace period

   -  – exceeded

   -  – flags

   -  – soft quota exceeded timestamp

   -  – soft inode quota

   -  – hard inode quota

   -  – soft length quota

   -  – hard length quota

   -  – soft size quota

   -  – hard size quota

   -  – soft real size quota

   -  – hard real size quota

**XATTR SECTION**

-  – line with xattr description

   -  – inode number

   -  – xattr name

   -  – xattr value

**POSIX ACL SECTION**

-  – line with acl description

   -  – inode number

   -  – acl type

   -  – user (file owner) permissions

   -  – group permissions

   -  – other permissions

   -  – permission mask

   -  – named permissions – list of objects:

      -  – permissions for user with uid

      -  – permissions for group with gid

**OPEN SECTION**

-  – number of chunkservers

-  – line with chunk server description

   -  – server ip

   -  – server port

   -  – server id

   -  – maintenance mode

**CHUNKSERVERS SECTION**

-  – line with open file description

   -  – session id

   -  – inode number

**CHUNKS SECTION**

-  – first available chunk number

-  – line with chunk description

   -  – chunk number

   -  – chunk version

   -  – “locked to” timestamp

   -  – archive flag

For MooseFS Supervisor
----------------------

– choose or switch leader master

**SYNOPSIS**

-  | 

-  
-  

| **DESCRIPTION**
| is the supervisor program of Moose File System. It is needed to start
  a completely new system or a system after a big crash. It can be also
  used to force select a new leader master.

**OPTIONS**

-  – print version information and exit

-  – print usage information and exit

-  – produce more verbose output

-  – dry run (print info, but do not change anything)

-  – force electing not synchronized follower; use this option to
   initialize a new system

-  – print info only about masters state

-  – try to switch current leader to given ip

-  – use given host to find your master servers (default: )

-  – use given port to connect to your master servers (default: )

For MooseFS Command Line Interface
----------------------------------

– CGI in TXT mode

**SYNOPSIS**

-  
-  
-  

**DESCRIPTION**

is a commandline counterpart to MooseFS’s CGI interface. All the
information available in CGI (except for graphs) can be obtained via CLI
using different “monitoring options”

**OPTIONS**:

-  – print help message

-  – force plain text format on tty devices

-  – do not resolve ip adresses (default when output device is not tty)

-  – field separator to use in plain text format on tty devices (forces
   -p)

-  – force 256-color terminal color codes

-  – force 8-color terminal color codes

-  master\_host – master address (default: mfsmaster)

-  master\_port – master client port (default: 9421)

-  – set frame charset to be displayed as table frames in ttymode

   -  – use simple ascii frames , , (default)

   -  – thick unicode frames

   -  – thin unicode frames

   -  – double unicode frames (dos style)

-  – sort data by column specified by (depends on data set)

-  – reverse sort order

-  – show data specified by (depends on data set)

**MONITORING OPTIONS**:

-  – show full master info

-  – show only masters states

-  – show only licence info

-  – show only general master (leader) info

-  – show only chunks info (goal/copies matrices)

-  – show only loop info (with messages)

-  – show connected chunk servers

-  – show connected metadata backup servers

-  – show hdd data

-  – show exports

-  – show active mounts

-  – show operation counters

-  – show quota info

-  – show master charts data

-  – show chunkserver charts data

**COMMANDS**:

-  – remove given chunkserver from list of active chunkservers

-  – send given chunkserver back to work (from grace state)

-  – switch selected chunkserver to maintenance mode

-  – switch selected chunkserver to standard mode (from maintenance
   mode)

-  – remove given session

**EXAMPLES**:

-  – shows table with chunk state matrix (number of chunks for each
   combination of valid copies and goal set by user) using extended
   terminal colors (256-colors) chunkservers

-  – shows table with all chunkservers using unicode thick frames

-  – shows current sessions (mounts) using plain text format and coma as
   a separator

For MooseFS CGI Server
----------------------

– start HTTP/CGI server for Moose File System monitoring

**SYNOPSIS**

-  
-  

**DESCRIPTION**

is a very simple HTTP server capable of running CGI scripts for Moose
File System monitoring.

**OPTIONS**

-  – print usage information and exit

-  – local address to listen on (default: any)

-  – port to listen on (default: )

-  – local path to use as HTTP document root (default is set up at
   configure time)

-  –run in foreground, don’t daemonize

-  – log requests on stderr

-  – how long to wait for lockfile (in seconds; default is 60 seconds)

is one of , , or . Default action is . The action will yeld one of two
responses: “” or “”.

For MooseFS Metalogger(s)
-------------------------

– start, restart or stop Moose File System metalogger process

**SYNOPSIS**

-  
-  
-  
-  

| **DESCRIPTION**
| is the metadata replication server of Moose File System. Depending on
  parameters it can start, restart or stop MooseFS metalogger process.
  Without any options it starts MooseFS metalogger, killing previously
  run process if lock file exists.

(or ’’ ) forces to reload all configuration files.

exists since 1.6.5 version of MooseFS; before this version was
responsible of logging metadata changes.

-  – print version information and exit

-  – print usage information and exit

-  – (**deprecated**, use start action instead) forcily run MooseFS
   metalogger process, without trying to kill previous instance (this
   option allows to run MooseFS metalogger if stale PID file exists)

-  – (**deprecated**, use stop action instead) stop MooseFS metalogger
   process

-  – specify alternative path of configuration file (default is in
   system configuration directory)

-  – log undefined configuration values (when default is assumed)

-  – run in foreground, don’t daemonize

-  – how long to wait for lockfile (default is 60 seconds)

is the one of , , , , or . Default action is unless (stop) or (start)
option is given. Note that and options are **deprecated**, likely to
disappear and parameter to become obligatory in MooseFS 1.7+.

**FILES**

-  – configuration file for MooseFS metalogger process; refer to
   mfsmetalogger.cfg(5) manual for details

-  – PID file of running MooseFS metalogger process (created in by
   MooseFS :math:`<` 1.6.9)

-  – lock file of running MooseFS metalogger process (created in data
   directory since MooseFS 1.6.9)

-  – MooseFS filesystem metadata change logs (backup of master change
   log files)

-  – Latest copy of complete file from MooseFS master.

-  – Latest copy of file from MooseFS master.

For MooseFS Chunkserver(s)
--------------------------

– start, restart or stop Moose File System chunkserver process

**SYNOPSIS**

-  
-  
-  

| **DESCRIPTION**
| is the data server of Moose File System.

**OPTIONS**

-  – print version information and exit

-  – print usage information and exit

-  – specify alternative path of configuration file (default is in
   system configuration directory)

-  – log undefined configuration values (when default is assumed)

-  – run in foreground, don’t daemonize

-  – how long to wait for lockfile (in seconds; default is 60 seconds)

is the one of , , , , or . Default action is . The test action will
yield one of two responses: “” or “”. The kill action will send a to the
currently running chunkserver process. or reload action forces to reload
all configuration files.

**FILES**

-  – configuration file for MooseFS chunkserver process; refer to
   mfschunkserver.cfg(5) manual for details

-  – list of directories (mountpoints) used for MooseFS storage; refer
   to mfshdd.cfg(5) manual for details

-  – lock file of running MooseFS chunkserver process (created in data
   directory)

-  – chunkserver charts state (created in data directory)

For MooseFS Client
------------------

– mount Moose File System

**SYNOPSIS**

-  
-  
-  

**DESCRIPTION**

Mount Moose File System.

General options:

-  , – display help and exit

-  – display version information and exit

FUSE options:

-  , – enable debug mode (implies )

-  – foreground operation

-  – disable multi-threaded operation

MooseFS options:

-  – loads file with additional mount options

-  , , – mount MFSMETA companion filesystem instead of primary MooseFS

-  – omit default mount options ()

-  – prompt for password (interactive version of )

-  , – connect with MooseFS master on (default is mfsmaster)

-  , – connect with MooseFS master on (default is 9421)

-  , – local address to use for connecting with master instead of
   default one

-  , – mount specified MooseFS directory (default is , i.e. whole
   filesystem)

-  – authenticate to MooseFS master with PASSWORD

-  – authenticate to MooseFS master using directly given MD5 (only if
   option is not specified)

-  – do not remember password in memory – more secure, but when session
   is lost then new session is created without password

-  – print some MooseFS-specific debugging information

-  – connection with master is done in background – with this option
   mount can be run without network (good for being run from fstab /
   init scripts etc.)

-  – sgid bit should be copied during mkdir operation (default depends
   on operating system)

-  – set sugid clear mode (see SUGID CLEAR MODES; default depends on
   operating system)

-  – set cache mode (see DATA CACHE MODES; default is AUTO)

-  – (deprecated) preserve file data in cache (equivalent to ’-o
   mfscachemode=YES’)

-  – set attributes cache timeout in seconds (default: 1.0)

-  – set extended attributes (xattr) cache timeout in seconds (default:
   30.0)

-  – set file entry cache timeout in seconds (default: 0.0, i.e. no
   cache)

-  – set directory entry cache timeout in seconds (default: 1.0)

-  – set negative entry cache timeout in seconds (default: 1.0)

-  – set supplementary groups cache timeout in seconds (default: 300.0)

-  – try to change limit of simultaneously opened file descriptors on
   startup (default: 100000)

-  – try to change nice level to specified value on startup (default:
   -19)

-  – specify write cache size in MiB (in range: 16..2048 - default: 250)

-  – specify number of retiries before I/O error is returned (default:
   30)

General mount options (see mount(8) manual):

-  – Mount file-system in read-write (default) or read-only mode
   respectively.

-  – Enable or disable suid/sgid attributes to work.

-  – Enable or disable character or block special device files
   interpretation.

-  – Allow or disallow execution of binaries.

**SUGID CLEAR MODE**

During attribute change file systems sometimes clear flags suid and/or
sgid. Behavior is different on different file systems. MFS tries to
mimic behavior of most popular file system on given operating systems.

-  – MFS will not change suid and sgid bit on chown

-  – clear suid and sgid on every chown - safest operation

-  – standard behavior in OS X and Solaris (chown made by unprivileged
   user clear suid and sgid)

-  – standard behavior in \*BSD systems (like in OSX, but only when
   something is really changed)

-  – standard behavior in most file systems on Linux (directories not
   changed, others: suid cleared always, sgid only when group exec bit
   is set)

-  – standard behavior in XFS on Linux (like EXT but directories are
   changed by unprivileged users)

**DATA CACHE MODES**

There are three cache modes: , and . Default option is and you shouldn’t
change it unless you really know what you are doing. In AUTO mode data
cache is managed automatically by .

-  , or – never allow files data to be kept in cache (safest but can
   reduce efficiency)

-  or – always allow files data to be kept in cache (dangerous)

-  – file cache is managed by automatically (should be very safe and
   efficient)

– perform MooseFS-specific operations

**SYNOPSIS**

-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  
-  

**DESCRIPTION**

-  and operate on object’s goal value, i.e. the number of copies in
   which all file data are stored. It means that file should survive
   failure of one less chunkservers than its goal value. Goal must be
   set between and (note that is strongly unadvised). prints current
   goal value of given object(s). option enables recursive mode, which
   works as usual for every given file, but for every given directory
   additionally prints current goal value of all contained objects
   (files and directories). changes current goal value of given
   object(s). If new value is specified in form, goal value is increased
   to for objects with lower goal value and unchanged for the rest.
   Similarly, if new value is specified as , goal value is decreased to
   for objects with higher goal value and unchanged for the rest. option
   enables recursive mode. These tools can be used on any file,
   directory or deleted (trash) file.

-  and are deprecated aliases for and respectively.

-  and operate on object’s trashtime value, i.e. the number of seconds
   the file is preserved in special trash directory before it’s finally
   removed from filesystem. Trashtime must be non-negative integer
   value. prints current trashtime value of given object(s). option
   enables recursive mode, which works as usual for every given file,
   but for every given directory additionally prints current trashtime
   value of all contained objects (files and directories). changes
   current trashtime value of given object(s). If new value is specified
   in form, trashtime value is increased to for objects with lower
   trashtime value and unchanged for the rest. Similarly, if new value
   is specified as , trashtime value is decreased to for objects with
   higher trashtime value and unchanged for the rest. option enables
   recursive mode. These tools can be used on any file, directory or
   deleted (trash) file.

-  and are deprecated aliases for and respectively.

-  , and tools are used to get, set or delete some extra attributes.
   Attributes are described below.

-  checks and prints number of chunks and number of chunk copies
   belonging to specified file(s). It can be used on any file, included
   deleted (trash).

-  prints location (chunkserver host and port) of each chunk copy
   belonging to specified file(s). It can be used on any file, included
   deleted (trash).

-  is extended, MooseFS-specific equivalent of command. It prints
   summary for each specified object (single file or directory tree). If
   you only want to see one parameter, then add one of show options (see
   SHOW OPTIONS)

-  deals with broken files (those which cause I/O errors on read
   operations) to make them partially readable. In case of missing chunk
   it fills missing parts of file with zeroes; in case of chunk version
   mismatch it sets chunk version known to to highest one found on
   chunkservers. Note: because in the second case content mismatch can
   occur in chunks with the same version, it’s advised to make a copy
   (not a snapshot!) and delete original file after “repairing”.

-  (equivalent of from MooseFS 1.5) appends a lazy copy of specified
   file(s) to specified snapshot file (“lazy” means that creation of new
   chunks is delayed to the moment one copy is modified). If multiple
   files are given, they are merged into one target file in the way that
   each file begins at chunk (64MB) boundary; padding space is left
   empty.

-  makes a “real” snapshot (lazy copy, like in case of ) of some
   object(s) or subtree (similarly to ). It’s atomic with respect to
   each argument separately. If points to already existing file, error
   will be reported unless (overwrite) option is given. Note: if is a
   directory, it’s copied as a whole; but if it’s followed by trailing
   slash, only directory content is copied.

-  , and tools are used to check, define and delete quotas. Quota is set
   on a directory. It can be set in one of 4 ways: for number of inodes
   inside the directory (total sum of the subtree’s inodes) with ,
   options, for sum of (logical) file lengths with , options, for sum of
   chunk sizes (not considering goals) with , options and for physical
   hdd space (more or less chunk sizes multiplied by goal of each chunk)
   with , options. Small letters set soft quota, capital letters set
   hard quota. and options in mean all kinds of quota. Quota behavior is
   described below.

-  tool can be used to find all occurrences (hard links) of given file
   in filesystem. Also can be used to find file by number of i-node. In
   case of searching by i-node tool has to be run in mfs mounted
   directory.

**GENERAL OPTIONS**

Most of mfstools use , , , , and options to select format of printed
numbers. causes to print exact numbers, uses binary prefixes (, , as
:math:`2^{10}`, :math:`2^{20}` etc.) while uses prefixes (, , as
:math:`10^3`, :math:`10^6` etc.). , and show plain numbers respectivaly
in kibis (binary kilo – 1024), mebis (binary mega – :math:`1024^2`) and
gibis (binary giga – :math:`1024^3`). The same can be achieved by
setting environment variable to: 0 (exact numbers), or (binary
prefixes), or ( prefixes), or (exact numbers and binary prefixes), or
(exact numbers and SI prefixes). The default is to print just exact
numbers.

**SHOW OPTIONS**

-  – show number of inodes

-  – show number of directories

-  – show number of files

-  – show number of chunks

-  – show length

-  – show size

-  – show realsize

**EXTRA ATTRIBUTES**

-  – This flag means, that particular object belongs to current user (
   and are equal to and values of accessing process). Only () sees the
   real and .

-  – This flag means, that standard file attributes such as , codegid, ,
   and so on won’t be stored in kernel cache. In MooseFS 1.5 this was
   the only behavior, and always prevented attributes from being stored
   in kernel cache, but in MooseFS 1.6 attributes can be cached, so in
   very rare ocassions it could be useful to turn it off.

-  – This flag is similar to above. It prevents directory entries from
   being cached in kernel.

**QUOTAS**

Quota is always set on a directory. Hard quota cannot be exceeded any
time. Soft quota can be exceeded for a period of time (7 days). Once a
quota is exceeded in a directory, user must go below the quota during
the next 7 days. If not, the soft quota for this particular directory
starts to behave like a hard quota. The 7 days period is global and
cannot currently be modified.

**INHERITANCE**

When new object is created in MooseFS, attributes such as , and extra
attributes are inherited from parent directory. So if you set i.e. “”
attribute and to in a directory then every new object created in this
directory will have set to and “” flag set. A newly created object
inherits always the current set of its parent’s attributes. Changing a
directory attribute does not affect its already created children. To
change an attribute for a directory and all of its children use “”
option.

MooseFS Configuration Files
===========================

For MooseFS Master Server(s)
----------------------------

****

– main configuration file for

| **DESCRIPTION**
| The file contains configuration of MooseFS master process.

| **SYNTAX**
| Syntax is:

Lines starting with character are ignored as comments.

| **OPTIONS**
| Configuration options:

-  – user to run daemon as

-  – group to run daemon as; optional value - if empty then default user
   group will be used

-  – name of process to place in syslog messages; default is

-  – whether to perform mlockall() to avoid swapping out process;
   default is 0, i.e. no

-  – nice level to run daemon with; default is -19; note: process must
   be started as root to increase priority, if setting of priority
   fails, process retains the nice level it started with

-  – set default umask for group and others (user has always 0); default
   is – block write for group and block all for others

-  – where to store metadata files and lock file

-  – alternate location/name of file

-  – alternate location/name of file

-  – alternate location/name of file (pro version only)

-  – number of metadata change log files (default is 50)

-  – number of previous metadata files to be kept (default is 1)

-  – how many seconds of change logs have to be preserved in memory
   (default is 1800; this sets the minimum, actual number may be a bit
   bigger due to logs being kept in 5k blocks; zero disables extra logs
   storage)

-  | – how many missing chunks will be stored in master (up to
   | bytes of memory will be allocated; default value is 100000)

-  – IP address to listen on for , s and s connections (\* means any)

-  – port to listen on for , s and s connections

-  – delay in seconds before next try to reconnect to if not connected
   (default is 5)

-  – timeout in seconds for connections (pro version only; default is
   10)

-  – local address to use for connecting with (pro version only; default
   is \*, i.e. default local address)

-  – IP address to listen on for connections ( means any)

-  – port to listen on for connections

-  – timeout in seconds for master-chunkserver connection (default is
   10)

-  – initial delay in seconds before starting replications (default is
   300)

-  – Chunks loop shouldn’t check more chunks per seconds than given
   number (default is 100000)

-  – Chunks loop shouldn’t be done in less seconds than given number
   (default is 300)

-  – Soft maximum number of chunks to delete on one (default is 10)

-  – Hard maximum number of chunks to delete on one (default is 25)

-  – Maximum number of chunks to replicate to one (default is 2,1,1,4 –
   see NOTES)

-  – Maximum number of chunks to replicate from one (default is 10,5,2,5
   – see NOTES)

-  – Threshold for chunkserver load (default is 100 – see NOTES)

-  – Threshold ratio for chunkserver load (default is 5.0 – see NOTES)

-  – Defines how long chunkservers will remain in ’grace’ mode (default
   is 900 – see NOTES)

-  – Maximum difference between space usage of chunkservers (deprecated,
   use instead)

-  – Maximum percentage difference between space usage of chunkservers
   (default is 1 = 1%)

-  – Length of priority queues (for endangered, undergoal etc. chunks –
   chunks that should be processed first – default is 1000000)

-  – IP address to listen on for client (mount) connections ( means any)

-  – port to listen on for client (mount) connections

-  – How long to sustain a disconnected client session (in seconds;
   default is 86400 = 1 day)

-  – Time limit in seconds for soft quota (default is 604800 = 7 days)

-  – Set atime modification mode (default is 0 = always modify atime –
   see NOTES)

**NOTES**

| Chunks in master are tested in a loop. Speed (or frequency) is
  regulated by two options and . First defines minimal time between
  iterations of the loop and second defines maximal number of chunk
  tests per second. Typically at the beginning, when number of chunks is
  small, time is constant, regulated by , but when number of chunks
  becomes bigger then time of loop can increase according to
| .

Example: is set to 300, is set to 100000 and there is 1000000 (one
million) chunks in the system. 1000000/100000 = 10, which is less than
300, so one loop iteration will take 300 seconds. With 1000000000 (one
billion) chunks the system needs 10000 seconds for one iteration of the
loop.

Deletion limits are defined as ’soft’ and ’hard’ limit. When number of
chunks to delete increases from loop to loop, current limit can be
temporary increased above soft limit, but never above hard limit.

Replication limits are divided into four cases:

-  first limit is for endangered chunks (chunks with only one copy)

-  second limit is for undergoal chunks (chunks with number of copies
   lower than specified goal)

-  third limit is for rebalance between servers with space usage around
   arithmetic mean

-  fourth limit is for rebalance between other servers (very low or very
   high space usage)

Usually first number should be grater than or equal to second, second
greater than or equal to third, and fourth greater than or equal to
third (1st :math:`>=` 2nd :math:`>=` 3rd :math:`<=` 4th). If one number
is given, then all limits are set to this number (for backward
compatibility).

| Whenever chunkserver load is higher than and
| times higher than average load, then chunkserver is switched into
  ’grace’ mode. Chunkserver stays in grace mode for seconds.

There are five values for (all other values are treated as ):

-  = Always modify atime for files, folders and symlinks.

-  = Always modify atime but only in case of files (do not modify atime
   in case of folders and symlinks).

-  = Modify atime only when it is lower than ctime or mtime and when
   current time is higher than ctime or mtime respectively, also modify
   atime when current atime is older than 24h. Do it for all objects
   during access (like “relatime” option in Linux).

-  = Same as above but only in case of files. In case of folders and
   symlinks do not modify atime.

-  = Never modify atime during access (like “noatime” option).

– MooseFS access control for s

**DESCRIPTION**

The file contains MooseFS access list for clients.

**SYNTAX**

Syntax is:

Lines starting with character are ignored as comments.

can be specified in several forms:

-  – all addresses

-  – single IP address

-  – IP class specified by network address and number of significant
   bits

-  – IP class specified by network address and mask

-  – IP range specified by from-to addresses (inclusive)

can be or path relative to MooseFS root; special value means MFSMETA
companion filesystem.

OPTIONS list:

-  , – export tree in read-only mode; this is default

-  , – export tree in read-write mode

-  – allows to mount any subdirectory of specified directory (similarly
   to NFS)

-  – allows reconnecting of already authenticated client from any IP
   address (the default is to check IP address on reconnect)

-  – disable testing of group access at level (it’s still done at level)
   – in this case “group” and “other” permissions are logically added;
   needed for supplementary groups to work ( receives only user primary
   group information)

-  – administrative privileges – currently: allow changing of quota
   values

-  – maps root () accesses to given user and group (similarly to option
   in NFS mounts); and can be given either as name or number; if no
   group is specified, ’s primary group is used. Names are resolved on
   side (see note below).

-  – like above but maps all non privileged users () accesses to given
   user and group (see notes below).

-  , – requires password authentication in order to access specified
   resource

-  – rejects access from clients older than specified

-  , – specify range in which goal can be set by users

-  , – specify range in which trashtime can be set by users

Default options are:

, , , , , .

**NOTES**

and names (if not specified by explicit / number) are resolved on host.

can be specified as number without time unit (number of seconds) or
combination of numbers with time units. Time units are: , , , , . Order
is important – less significant time units can’t be defined before more
significant time units. Time units are case insensitive.

Option works in MooseFS in different way than in NFS, because MooseFS is
using FUSE’s “” option. When option is used, users see all objects with
equal to mapped as their own and all other as root’s objects. Similarly
objects with equal to mapped are seen as objects with current user’s
primary group and all other objects as objects with group 0 (usually
wheel). With option set attribute cache in kernel is always turned off.

**EXAMPLES**

::

        *                    /          ro
        192.168.1.0/24       /          rw
        192.168.1.0/24       /          rw,alldirs,maproot=0,password=passcode
        10.0.0.0-10.0.0.5    /test      rw,maproot=nobody,password=test
        10.1.0.0/255.255.0.0 /public    rw,mapall=1000:1000
        10.2.0.0/16          /          rw,alldirs,maproot=0,mintrashtime=2h30m,maxtrashtime=2w
                    

– MooseFS network topology definitions

**DESCRIPTION**

The file contains assignments of IP addresses into network locations
(usually switch numbers). This file is optional. If your network has one
switch or decreasing traffic between switches is not necessary then
leave this file empty.

**SYNTAX**

Syntax is:

Lines starting with character are ignored as comments.

can be specified in several forms:

-  – all addresses

-  – single IP address

-  – IP class specified by network address and bits number

-  – IP class specified by network address and mask

-  – IP range specified by from-to addresses (inclusive)

can be specified as any positive 32-bit numer.

| **NOTES**
| If one IP belongs to more than one definition then last definition is
  used.

As for now distance between switches is constant. So distance between
machines is calculated as: 0 when IP numbers are the same, 1 when IP
numbers are different, but switch numbers are the same and 2 when switch
numbers are different

Distances are used only to sort chunkservers during read and write
operations. New chunks are still created randomly. Also rebalance
routines do not take distances into account.

For MooseFS Metalogger(s)
-------------------------

codemfsmetalogger.cfg – configuration file for mfsmetalogger

**DESCRIPTION**

The file contains configuration of MooseFS process.

**SYNTAX**

Syntax is:

Lines starting with character are ignored as comments.

**OPTIONS**

Configuration options:

-  – where to store metadata files

-  – (deprecated) daemon lock/pid file

-  – user to run daemon as

-  – group to run daemon as (optional – if empty then default user group
   will be used)

-  – name of process to place in syslog messages (default is
   mfsmetalogger)

-  – whether to perform mlockall() to avoid swapping out mfsmetalogger
   process (default is 0, i.e. no)

-  – nice level to run daemon with (default is -19 if possible; note:
   process must be started as root to increase priority)

-  – number of metadata change log files (default is 50)

-  – number of previous metadata files to be kept (default is 3)

-  – metadata download frequency in hours (default is 24, at most /2)

-  – address of MooseFS master host to connect with (default is
   mfsmaster)

-  – number of MooseFS master port to connect with (default is 9420)

-  – delay in seconds before trying to reconnect to master after
   disconnection (default is 30)

-  – timeout (in seconds) for master connections (default is 60)

For MooseFS Chunkservers
------------------------

– main configuration file for mfschunkserver

**DESCRIPTION**

The file contains configuration of MooseFS process.

**SYNTAX**

Syntax is:

Lines starting with character are ignored as comments.

**OPTIONS**

Configuration options:

-  – user to run daemon as

-  – group to run daemon as; optional value – if empty then default user
   group will be used

-  – name of process to place in syslog messages; default is

-  – whether to perform mlockall() to avoid swapping out process;
   default is , i.e. no

-  – nice level to run daemon with; default is ; note: process must be
   started as root to increase priority, if setting of priority fails,
   process retains the nice level it started with

-  – set default umask for group and others (user has always 0); default
   is – block write for group and block all for others

-  – where to store daemon lock file

-  – alternate location/name of file

-  – chunk test period in seconds; default is

-  – how much space should be left unused on each hard drive; number
   format: ; default is ; examples: , , , etc.

-  – percent of total work time the chunkserver is allowed to spend on
   hdd space rebalancing; default is 20

-  , – how many i/o errors () to tolerate in given amount of seconds ()
   on a single hard drive; if the number of errors exceeds this setting,
   the offending hard drive will be marked as damaged; defaults are 2
   and 600

-  – enables/disables fsync before chunk closing; deafult is 0 (off)

-  , – maximum number of active workers and maximum number of idle
   workers; defaults are 150 and 40

-  – local address to use for master connections; default is , i.e.
   default local address

-  – MooseFS master host, IP is allowed only in single-master
   installations; default is

-  – MooseFS master command port; default is

-  – MooseFS master control port; default is

-  – timeout in seconds for master connections; default is

-  – delay in seconds before trying to reconnect to master after
   disconnection (default is 5)

-  – local address to use for connecting with master (default is , i.e.
   default local address)

-  – IP address to listen on for client (mount) connections ( means any)

-  – port to listen on for client (mount) connections (default is )

-  – timeout (in seconds) for client (mount) connections (default is )

– list of MooseFS storage directories for mfschunkserver

**DESCRIPTION**

The file contains list of directories (mountpoints) used for MooseFS
storage.

**SYNTAX**

Syntax is:

Lines starting with character are ignored as comments.

means this directory (hard drive) is “marked for removal” and all data
will be replicated to other hard drives, usually on other chunkservers

is path to the mounting point of storage directory, usually a single
hard drive.

is optional space limit, that allows to set one of two values: how much
space should be left unused on this device or how much space is to be
used on this device. Definition format: , positive value means how much
space to use, negative value means how much space should be left unused.

Frequently Asked Questions
==========================

What average write/read speeds can we expect?
---------------------------------------------

Aside from common (for most filesystems) factors like: block size and
type of access (sequential or random), in MooseFS the speeds depend also
on hardware performance. Main factors are hard drives performance and
network capacity and topology (network latency). The better the
performance of the hard drives used and the better throughput of the
network, the higher performance of the whole system.

Does the goal setting influence writing/reading speeds?
-------------------------------------------------------

Generally speaking, it does not. In case of reading a file, goal higher
than one may in some cases help speed up the reading operation, i. e.
when two clients access a file with goal two or higher, they may perform
the read operation on different copies, thus having all the available
throughtput for themselves. But in average the goal setting does not
alter the speed of the reading operation in any way.

Similarly, the writing speed is negligibly influenced by the goal
setting. Writing with goal higher than two is done chain-like: the
client send the data to one chunk server and the chunk server
simultaneously reads, writes and sends the data to another chunk server
(which may in turn send them to the next one, to fulfill the goal). This
way the client’s throughtput is not overburdened by sending more than
one copy and all copies are written almost simultaneously. Our tests
show that writing operation can use all available bandwidth on client’s
side in 1Gbps network.

Are concurrent read and write operations supported?
---------------------------------------------------

All read operations are parallel – there is no problem with concurrent
reading of the same data by several clients at the same moment. Write
operations are parallel, execpt operations on the same chunk (fragment
of file), which are synchronized by Master server and therefore need to
be sequential.

How much CPU/RAM resources are used?
------------------------------------

In our environment (ca. 1 PiB total space, 36 million files, 6 million
folders distributed on 38 million chunks on 100 machines) the usage of
chunkserver CPU (by constant file transfer) is about 15-30% and
chunkserver RAM usually consumes in between 100MiB and 1GiB (dependent
on amount of chunks on each chunk server). The master server consumes
about 50% of modern 3.3 GHz CPU (ca. 5000 file system operations per
second, of which ca. 1500 are modifications) and 12GiB RAM. CPU load
depends on amount of operations and RAM on the total number of files and
folders, not the total size of the files themselves. The RAM usage is
proportional to the number of entries in the file system because the
master server process keeps the entire metadata in memory for
performance. HHD usage on our master server is ca. 22 GB.

Is it possible to add/remove chunkservers and disks on the fly?
---------------------------------------------------------------

You can add/remove chunk servers on the fly. But keep in mind that it is
not wise to disconnect a chunk server if this server contains the only
copy of a chunk in the file system (the CGI monitor will mark these in
orange). You can also disconnect (change) an individual hard drive. The
scenario for this operation would be:

#. Mark the disk(s) for removal (see How to mark a disk for removal?)

#. Reload the chunkserver process

#. Wait for the replication (there should be no “undergoal” or “missing”
   chunks marked in yellow, orange or red in CGI monitor)

#. Stop the chunkserver process

#. Delete entry(ies) of the disconnected disk(s) in

#. Stop the chunkserver machine

#. Remove hard drive(s)

#. Start the machine

#. Start the chunkserver process

If you have hotswap disk(s) you should follow these:

#. Mark the disk(s) for removal (see How to mark a disk for removal?)

#. Reload the chunkserver process

#. Wait for the replication (there should be no “undergoal” or “missing”
   chunks marked in yellow, orange or red in CGI monitor)

#. Delete entry(ies) of the disconnected disk(s) in

#. Reload the chunkserver process

#. Unmount disk(s)

#. Remove hard drive(s)

If you follow the above steps, work of client computers won’t be
interrupted and the whole operation won’t be noticed by MooseFS users.

How to mark a disk for removal?
-------------------------------

When you want to mark a disk for removal from a chunkserver, you need to
edit the chunkserver’s configuration file and put an asterisk ’’ at the
start of the line with the disk that is to be removed. For example, in
this we have marked “” for removal:

::

        /mnt/hda
        /mnt/hdb
        /mnt/hdc
        */mnt/hdd
        /mnt/hde 
            

After changing the mfshdd.cfg you need to reload chunkserver (on Linux
Debian/Ubuntu: ).

Once the disk has been marked for removal and the chunkserver process
has been restarted, the system will make an appropriate number of copies
of the chunks stored on this disk, to maintain the required “goal”
number of copies.

Finally, before the disk can be disconnected, you need to confirm there
are no “undergoal” chunks on the other disks. This can be done using the
CGI Monitor. In the “Info” tab select “Regular chunks state matrix”
mode.

My experience with clustered filesystems is that metadata operations are quite slow. How did you resolve this problem?
----------------------------------------------------------------------------------------------------------------------

During our research and development we also observed the problem of slow
metadata operations. We decided to aleviate some of the speed issues by
keeping the file system structure in RAM on the metadata server. This is
why metadata server has increased memory requirements. The metadata is
frequently flushed out to files on the master server.

Additionally, in MooseFS (non-Pro), the Metadata logger server(s) also
frequently receive updates to the metadata structure and write these to
their file systems.

In Pro version metaloggers are optional, because master followers are
keeping synchronised with leader master. They’re also saving metadata to
the hard disk.

What does value of directory size mean on MooseFS? It is different than standard Linux output. Why?
---------------------------------------------------------------------------------------------------

Folder size has no special meaning in any filesystem, so our development
team decided to give there extra information. The number represents
total length of all files inside (like in ) displayed in exponential
notation.

You can “translate” the directory size by the following way:

There are 7 digits: . To translate this notation to number of bytes, use
the following expression:

Where :

-  
-  
-  
-  
-  

:

To translate the following entry:

::

        drwxr-xr-x 164 root root 2010616 May 24 11:47 test
                                 xAAAABB
            

Folder size should be read as 106.16 MiB.

When , the number might be smaller:

:

Folder size means 102 Bytes.

When I perform on a filesystem the results are different from what I would expect taking into account actual sizes of written files.
------------------------------------------------------------------------------------------------------------------------------------

Every chunkserver sends its own disk usage increased by 256MB for each
used partition/hdd, and the master sends a sum of these values to the
client as total disk usage. If you have 3 chunkservers with 7 hdd each,
your disk usage will be increased by 3\*7\*256MB (about 5GB).

The other reason for differences is, when you use disks exclusively for
MooseFS on chunkservers will show correct disk usage, but if you have
other data on your MooseFS disks will count your own files too.

If you want to see the actual space usage of your MooseFS files, use
command.

Can I keep source code on MooseFS? Why do small files occupy more space than I would have expected?
---------------------------------------------------------------------------------------------------

The system was initially designed for keeping large amounts (like
several thousands) of very big files (tens of gigabytes) and has a
hard-coded chunk size of 64MiB and block size of 64KiB. Using a
consistent block size helps improve the networking performance and
efficiency, as all nodes in the system are able to work with a single
’bucket’ size. That’s why even a small file will occupy 64KiB plus
additionally 4KiB of checksums and 1KiB for the header.

The issue regarding the occupied space of a small file stored inside a
MooseFS chunk is really more significant, but in our opinion it is still
negligible. Let’s take 25 million files with a goal set to 2. Counting
the storage overhead, this could create about 50 million 69 KiB chunks,
that may not be completely utilized due to internal fragmentation
(wherever the file size was less than the chunk size). So the overall
wasted space for the 50 million chunks would be approximately 3.2TiB. By
modern standards, this should not be a significant concern. A more
typical, medium to large project with 100,000 small files would consume
at most 13GiB of extra space due to block size of used file system.

So it is quite reasonable to store source code files on a MooseFS
system, either for active use during development or for long term
reliable storage or archival purposes.

Perhaps the larger factor to consider is the comfort of developing the
code taking into account the performance of a network file system. When
using MooseFS (or any other network based file system such as NFS, CIFS)
for a project under active development, the network filesystem may not
be able to perform file IO operations at the same speed as a directly
attached regular hard drive would.

Some modern integrated development environments (IDE), such as Eclipse,
make frequent IO requests on several small workspace metadata files.
Running Eclipse with the workspace folder on a MooseFS file system (and
again, with any other networked file system) will yield slightly slower
user interface performance, than running Eclipse with the workspace on a
local hard drive.

You may need to evaluate for yourself if using MooseFS for your working
copy of active development within an IDE is right for you.

In a different example, using a typical text editor for source code
editing and a version control system, such as Subversion, to check out
project files into a MooseFS file system, does not typically resulting
any performance degradation. The IO overhead of the network file system
nature of MooseFS is offset by the larger IO latency of interacting with
the remote Subversion repository. And the individual file operations
(open, save) do not have any observable latencies when using simple text
editors (outside of complicated IDE products).

A more likely situation would be to have the Subversion repository files
hosted within a MooseFS file system, where the svnserver or Apache
:math:`+` mod\_svn would service requests to the Subversion repository
and users would check out working sandboxes onto their local hard
drives.

Do Chunkservers and Metadata Server do their own checksumming?
--------------------------------------------------------------

Chunk servers do their own checksumming. Overhead is about 4B per a
64KiB block which is 4KiB per a 64MiB chunk. Metadata servers don’t. We
thought it would be CPU consuming. We recommend using ECC RAM modules.

What resources are required for the Master Server?
--------------------------------------------------

The most important factor is RAM of machine, as the full file system
structure is cached in RAM for speed. Besides RAM machine needs some
space on HDD for main metadata file together with incremental logs. The
size of the metadata file is dependent on the number of files (not on
their sizes). The size of incremental logs depends on the number of
operations per hour, but length (in hours) of this incremental log is
configurable.

When I delete files or directories, the MooseFS free space size doesn’t change. Why?
------------------------------------------------------------------------------------

MooseFS does not immediately erase files on deletion, to allow you to
revert the delete operation. Deleted files are kept in the trash bin for
the configured amount of time (default: 24h / 86400 seconds) before they
are deleted.

You can configure for how long files are kept in trash and empty the
trash manually (to release the space).

You cant mount the trash e.g. in the following way: First of all, create
the directory to mount

::

        # mkdir /mnt/mfsmeta
            

Then, mount (like normally, but with parameter:

::

        # mfsmount -H master.host.name -m /mnt/mfsmeta
            

or:

::

        # mfsmount -H master.host.name -o mfsmeta /mnt/mfsmeta
            

Then, go into trash subdirectory:

::

        # cd /mnt/mfsmeta/trash
            

You can see 4096 sub-trashes in this directory (named ). The reason of
divide the trash into sub-trashes is a huge amount of files in trash on
big instances. In such case, commands like or even are not able to
functionate properly. So since MooseFS 3, deleted files are located
inside these directories (sub-trashes). The best way to locate a file
you are looking for is to use command.

If you use MooseFS 3 and you want to see the old trash structure, known
from MooseFS 2 (because you e.g. don’t have a lot of files in trash and
you like old, simple structure), you should mount the trash with a
specific parameter, e.g.:

::

        # mfsmount -H master.host.name -o mfsmeta,mfsflattrash /mnt/mfsmeta
            

If you want to delete files from trash on MooseFS 3 with new trash
structure, you should combine and commands together, e.g.:

::

        # mkdir /mnt/mfsmeta
        # mfsmount -H master.host.name -o mfsmeta /mnt/mfsmeta
        # cd /mnt/mfsmeta/trash
        # find . -type f -exec rm {} \;
            

In case you want to delete files from trash with old structure on
MooseFS 3, just issue command like above, but firstly mount it with
parameter, e.g.:

::

        # mkdir /mnt/mfsmeta
        # mfsmount -H master.host.name -o mfsmeta,mfsflattrash /mnt/mfsmeta
        # cd /mnt/mfsmeta/trash
        # rm *
            

The time of storing a deleted file can be verified by the command and
changed with .

When I added a third server as an extra chunkserver, it looked like the system started replicating data to the 3rd server even though the file goal was still set to 2.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Yes. Disk usage balancer uses chunks independently, so one file could be
redistributed across all of your chunkservers.

Is MooseFS 64bit compatible?
----------------------------

Yes!

Can I modify the chunk size?
----------------------------

No. File data is divided into fragments (chunks) with a maximum of 64MiB
each. The value of 64 MiB is hard coded into system so you cannot modify
its size. We based the chunk size on real-world data and determined it
was a very good compromise between number of chunks and speed of
rebalancing / updating the filesystem. Of course if a file is smaller
than 64 MiB it occupies less space.

In the systems we take care of, several file sizes significantly exceed
100GB with no noticable chunk size penalty.

How do I know if a file has been successfully written to MooseFS?
-----------------------------------------------------------------

Let’s briefly discuss the process of writing to the file system and what
programming consequences this bears.

In all contemporary filesystems, files are written through a buffer
(write cache). As a result, execution of the write command itself only
transfers the data to a buffer (cache), with no actual writing taking
place. Hence, a confirmed execution of the write command does not mean
that the data has been correctly written on a disk. It is only with the
invocation and completion of the fsync (or close) command that causes
all data kept within the buffers (cache) to get physically written out.
If an error occurs while such buffer-kept data is being written, it
could cause the fsync (or close) command to return an error response.

The problem is that a vast majority of programmers do not test the close
command status (which is generally a very common mistake). Consequently,
a program writing data to a disk may “assume” that the data has been
written correctly from a success response from the write command, while
in actuality, it could have failed during the subsequent close command.

In network filesystems (like MooseFS), due to their nature, the amount
of data “left over” in the buffers (cache) on average will be higher
than in regular file systems. Therefore the amount of data processed
during execution of the close or fsync command is often significant and
if an error occurs while the data is being written [from the close or
fsync command], this will be returned as an error during the execution
of this command. Hence, before executing close, it is recommended
(especially when using MooseFS) to perform an fsync operation after
writing to a file and then checking the status of the result of the
fsync operation. Then, for good measure, also check the return status of
close as well.

NOTE! When stdio is used, the fflush function only executes the “write”
command, so correct execution of fflush is not sufficient to be sure
that all data has been written successfully – you should also check the
status of fclose.

The above problem may occur when redirecting a standard output of a
program to a file in shell. Bash (and many other programs) do not check
the status of the close execution. So the syntax of “” type may wrap up
successfully in shell, while in fact there has been an error in writing
out the “” file. You are strongly advised to avoid using the above shell
output redirection syntax when writing to a MooseFS mount point. If
necessary, you can create a simple program that reads the standard input
and writes everything to a chosen file, where this simple program would
correctly employ the appropriate check of the result status from the
fsync command. For example, “”, where is the name of your writing
program instead of .

Please note that the problem discussed above is in no way exceptional
and does not stem directly from the characteristics of MooseFS itself.
It may affect any system of files – network type systems are simply more
prone to such difficulties. Technically speaking, the above
recommendations should be followed at all times (also in cases where
classic file systems are used).

What are limits in MooseFS (e.g. file size limit, filesystem size limit, max number of files, that can be stored on the filesystem)?
------------------------------------------------------------------------------------------------------------------------------------

-  The maximum file size limit in MooseFS is :math:`2^{57}` bytes = 128
   PiB.

-  The maximum filesystem size limit is :math:`2^{64}` bytes = 16 EiB =
   16 384 PiB

-  The maximum number of files, that can be stored on one MooseFS
   instance is :math:`2^{31}` – over 2.1 bln.

Can I set up HTTP basic authentication for the mfscgiserv?
----------------------------------------------------------

is a very simple HTTP server written just to run the MooseFS CGI
scripts. It does not support any additional features like HTTP
authentication. However, the MooseFS CGI scripts may be served from
another full-featured HTTP server with CGI support, such as lighttpd or
Apache. When using a full-featured HTTP server such as Apache, you may
also take advantage of features offered by other modules, such as HTTPS
transport. Just place the CGI and its data files (, , , , , , ) under
chosen . If you already have an HTTP server instance on a given host,
you may optionally create a virtual host to allow access to the MooseFS
CGI Monitor through a different hostname or port.

Can I run a mail server application on MooseFS? Mail server is a very busy application with a large number of small files – will I not lose any files?
------------------------------------------------------------------------------------------------------------------------------------------------------

You can run a mail server on MooseFS. You won’t lose any files under a
large system load. When the file system is busy, it will block until its
operations are complete, which will just cause the mail server to slow
down.

Are there any suggestions for the network, MTU or bandwidth?
------------------------------------------------------------

We recommend using jumbo-frames [4]_ (MTU=9000). With a greater amount
of chunkservers, switches should be connected through optical fiber or
use aggregated links [5]_.

Does MooseFS support supplementary groups?
------------------------------------------

Yes.

Does MooseFS support file locking?
----------------------------------

Yes, since MooseFS 3.0.

Is it possible to assign IP addresses to chunk servers via DHCP?
----------------------------------------------------------------

Yes, but we highly recommend setting “DHCP reservations” based on MAC
addresses.

Some of my chunkservers utilize 90% of space while others only 10%. Why does the rebalancing process take so long?
------------------------------------------------------------------------------------------------------------------

Our experiences from working in a production environment have shown that
aggressive replication is not desirable, as it can substantially slow
down the whole system. The overall performance of the system is more
important than equal utilization of hard drives over all of the chunk
servers. By default replication is configured to be a non-aggressive
operation. At our environment normally it takes about 1 week for a new
chunkserver to get to a standard hdd utilization. Aggressive replication
would make the whole system considerably slow for several days.

Replication speeds can be adjusted on master server startup by setting
these two options:

-  | 
   | Maximum number of chunks to replicate to one chunkserver (default
     is ).
   | One number is equal to four same numbers separated by colons.

   -  First limit is for endangered chunks (chunks with only one copy)

   -  Second limit is for undergoal chunks (chunks with number of copies
      lower than specified goal)

   -  Third limit is for rebalance between servers with space usage
      around arithmetic mean

   -  Fourth limit is for rebalance between other servers (very low or
      very high space usage)

   Usually first number should be grater than or equal to second, second
   greater than or equal to third, and fourth greater than or equal to
   third (1st :math:`>=` 2nd :math:`>=` 3rd :math:`<=` 4th)

-  | 
   | Maximum number of chunks to replicate from one chunkserver (default
     is ).
   | One number is equal to four same numbers separated by colons. '
     Limit groups are the same as in write limit, also relations between
     numbers should be the same as in write limits (1st :math:`>=` 2nd
     :math:`>=` 3rd :math:`<=` 4th)

Tuning these in your environment requires some experiments.

I have a Metalogger running – should I make additional backup of the metadata file on the Master Server?
--------------------------------------------------------------------------------------------------------

Yes, it is highly recommended to make additional backup of the metadata
file. This provides a worst case recovery option if, for some reason,
the metalogger data is not useable for restoring the master server (for
example the metalogger server is also destroyed).

The master server flushes metadata kept in RAM to the binary file every
hour on the hour (xx:00). So a good time to copy the metadata file is
every hour on the half hour (30 minutes after the dump). This would
limit the amount of data loss to about 1.5h of data. Backing up the file
can be done using any conventional method of copying the metadata file –
cp, scp, rsync, etc.

After restoring the system based on this backed up metadata file the
most recently created files will have been lost. Additionally files,
that were appended to, would have their previous size, which they had at
the time of the metadata backup. Files that were deleted would exist
again. And files that were renamed or moved would be back to their
previous names (and locations). But still you would have all of data for
the files created in the X past years before the crash occurred.

In MooseFS Pro version, master followers flush metadata from RAM to the
hard disk once an hour. The leader master downloads saved metadata from
followers once a day.

I think one of my disks is slower / damaged. How should I find it?
------------------------------------------------------------------

In the CGI monitor go to the “Disks” tab and choose “switch to hour” in
“I/O stats” column and sort the results by “write” in “max time” column.
Now look for disks which have a significantly larger write time. You can
also sort by the “fsync” column and look at the results. It is a good
idea to find individual disks that are operating slower, as they may be
a bottleneck to the system.

It might be helpful to create a test operation, that continuously copies
some data to create enough load on the system for there to be observable
statisics in the CGI monitor. On the “Disks” tab specify units of
“minutes” instead of hours for the “I/O stats” column.

Once a “bad” disk has been discovered to replace it follow the usual
operation of marking the disk for removal, and waiting until the color
changes to indicate that all of the chunks stored on this disk have been
replicated to achieve the sufficient goal settings.

How can I find the master server PID?
-------------------------------------

Issue the following command:

Web interface shows there are some copies of chunks with goal 0. What does it mean?
-----------------------------------------------------------------------------------

This is a way to mark chunks belonging to the non-existing (i.e.
deleted) files. Deleting a file is done asynchronously in MooseFS.
First, a file is removed from metadata and its chunks are marked as
unnecessary (). Later, the chunks are removed during an “idle” time.
This is much more efficient than erasing everything at the exact moment
the file was deleted.

Unnecessary chunks may also appear after a recovery of the master
server, if they were created shortly before the failure and were not
available in the restored metadata file.

Is every error message reported by a serious problem?
-----------------------------------------------------

No. writes every failure encountered during communication with
chunkservers to the syslog. Transient communication problems with the
network might cause IO errors to be displayed, but this does not mean
data loss or that will return an error code to the application. Each
operation is retried by the client () several times and only after the
number of failures (reported as ) reaches a certain limit (typically
30), the error is returned to the application that data was not
read/saved.

Of course, it is important to monitor these messages. When messages
appear more often from one chunkserver than from the others, it may mean
there are issues with this chunkserver – maybe hard drive is broken,
maybe network card has some problems – check its charts, hard disk
operation times, etc. in the CGI monitor.

Note: in examples below means IP address of chunkserver. In version
:math:`<` 2.0.42 chunkserver IP is written in hexadecimal format. In
version :math:`>=` 2.0.42 IP is “human-readable”.

| What does
| message mean?

This means that Zth try to write the chunk was not successful and
writing of Y blocks, sent to the chunkserver, was not confirmed. After
reconnecting these blocks would be sent again for saving. The limit of
trials is set by default to 30.

This message is for informational purposes and doesn’t mean data loss.

| What does
| message mean?

This means that Zth try to read the chunk was not successful and system
will try to read the block again. If value of Z equals 1 it is a
transitory problem and you should not worry about it. The limit of
trials is set by default to 30.

How do I verify that the MooseFS cluster is online? What happens with when the master server goes down?
-------------------------------------------------------------------------------------------------------

When the master server goes down while is already running, doesn’t
disconnect the mounted resource, and files awaiting to be saved would
stay quite long in the queue while trying to reconnect to the master
server. After a specified number of tries they eventually return EIO –
“input/output error”. On the other hand it is not possible to start when
the master server is offline.

There are several ways to make sure that the master server is online, we
present a few of these below. Check if you can connect to the TCP port
of the master server (e.g. socket connection test). In order to assure
that a MooseFS resource is mounted it is enough to check the inode
number – MooseFS root will always have inode equal to 1. For example if
we have MooseFS installation in then command (in Linux) will show:

::

            $ stat /mnt/mfs
              File: `/mnt/mfs'
              Size: xxxxxx      Blocks: xxx     IO Block: 4096   directory
            Device: 13h/19d     Inode: 1        Links: xx
            (...)
            

Additionaly creates a virtual hidden file in the root mounted folder.
For example, to get the statistics of when MooseFS is mounted we can
this file, eg.:

::

            $ cat /mnt/mfs/.stats
            fuse_ops.statfs: 241
            fuse_ops.access: 0
            fuse_ops.lookup-cached: 707553
            fuse_ops.lookup: 603335
            fuse_ops.getattr-cached: 24927
            fuse_ops.getattr: 687750
            fuse_ops.setattr: 24018
            fuse_ops.mknod: 0
            fuse_ops.unlink: 23083
            fuse_ops.mkdir: 4
            fuse_ops.rmdir: 1
            fuse_ops.symlink: 3
            fuse_ops.readlink: 454
            fuse_ops.rename: 269
            (...)
            

If you want to be sure that master server properly responds you need to
try to read the goal of any object, e.g. of the root folder:

::

            $ mfsgetgoal /mnt/mfs
            /mnt/mfs: 2
            

If you get a proper goal of the root folder, you can be sure that the
master server is up and running.

.. [1]
   You can read more about FUSE at http://fuse.sourceforge.net

.. [2]
   https://github.com/glk/fuse-freebsd

.. [3]
   http://osxfuse.github.com

.. [4]
   https://en.wikipedia.org/wiki/Jumbo_frame

.. [5]
   https://en.wikipedia.org/wiki/Link_aggregation

.. |image| image:: images/moosefs.png
   :width: 20.0%
.. |image| image:: images/read_mfs.png
   :width: 80.0%
.. |image| image:: images/write_mfs.png
   :width: 80.0%
.. |image| image:: images/diagram_without_labels.png
.. |image| image:: images/cgi_nolabels.png
   :width: 100.0%
.. |image| image:: images/cgi_labelsAB.png
   :width: 100.0%
.. |image| image:: images/diagram_with_labels.png
.. |image| image:: images/cgi_resources1.png
   :width: 100.0%
.. |image| image:: images/cgi_labelsAB_data.png
   :width: 100.0%
.. |image| image:: images/cgi_resources2.png
   :width: 100.0%
.. |image| image:: images/diagram_A_B_v2.png
.. |image| image:: images/diagram_ssd_hdd_v2.png
.. |image| image:: images/diagram_A_B_ssd_hdd_v2.png

