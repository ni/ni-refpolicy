National Instruments SELinux Reference Policy
======

Introduction
------

This repository contains the NI reference policy for SELinux, based on the
Tresys Technologies SELinux reference policy version 2.20140311 with patches
from OpenEmbedded and National Instruments.  The original Tresys reference
policy is available on GitHub
[here](https://github.com/TresysTechnology/refpolicy), and additional
documentation on it is available on [the corresponding
wiki](https://github.com/TresysTechnology/refpolicy/wiki).

Security Enhanced Linux (SELinux) is a mandatory access control (MAC) based
system that uses a security policy to explicitly specify the actions that each
component of the system is allowed to perform.

For example, the `cat` executable should be allowed to access files that the
user has permissions to read, but it should not be allowed to open network
sockets -- something a malicious or exploited version of `cat` might try to do.
If it tries to do so, SELinux intercepts the system call, determines that the
access is not permitted by the policy, denies the access, and logs the denial.


National Instruments support for SELinux
------

Via the package repository, National Instruments provides the libraries and
tools needed to install SELinux to your real-time target.  In addition,
National Instruments provides a reference policy that covers the behavior of
some of the NI-specific services and options on our real-time targets.
However, National Instruments does not provide support for installing,
configuring, or using SELinux beyond this documentation.  If you encounter
problems with the SELinux implementation on your target, National Instruments
recommends seeking external support or removing SELinux.

*Note: You must take further actions beyond enabling SELinux to legitimately
increase the security of your system.  SELinux requires a customized policy to
tell it what actions the different parts of your system are allowed to perform,
and crafting this policy requires detailed knowledge of the low-level behavior
of your application and of a Linux system overall.  For customers who are not
already skilled in SELinux policy work, we recommend working with a third party
to develop a policy for your application (e.g. Tresys Technologies, upon whose
SELinux reference policy the NI reference policy is based).*


Installing SELinux with the default NI reference policy
------

To get started with SELinux on your NI real-time target, complete the following
steps to install the SELinux stack with the NI reference policy.

1. Ensure your target is in a running state with both SSH and console out
enabled.  You can configure these options in NI MAX.

2. Run `opkg update` and then `opkg install packagegroup-ni-selinux`.  This
packagegroup contains the SELinux userspace tools and libraries, as well as a
compiled version of the NI reference policy contained in this repository.

3. Edit `/etc/selinux/config` and change the `DEFAULT_POLICY` to be `standard`
rather than `mls`, and change `SELINUX=enforcing` to `SELINUX=permissive` -- you
can put SELinux into enforcing mode once your policy has been appropriately
tweaked.

4. Run `fw_setenv bootdelay 5` (where the number represents the number of
seconds for which the target will prompt to enter the bootloader before
beginning the boot process).  This will enable verbose output to the serial
console, and it will enable you to easily switch back to safe mode in the event
that you get your run mode into a bad state.

5. Add boot arguments to the run mode kernel to enable SELinux:

  * (ARM-based targets):
From the command line, run `fw_setenv othbootargs selinux=1 security=selinux`.

  * (Intel x64-based targets):
Edit `/boot/runmode/bootimage.cfg`.  At the end of the line defining the
`otherbootargs` variable, add `selinux=1 security=selinux`.

6. Restart your target.  On startup, the system relabels the filesystems and
then restarts.  On the second startup, the system loads SELinux in permissive
mode and displays SELinux warnings on the console.


Customizing the NI reference policy for your application
------

By default, the NI reference policy runs LabVIEW (and any VIs that run within
it) as an unprivileged user, blocking it from performing many legitimate
activities.  You must customize the reference policy to give LabVIEW a set of
access privileges that make sense for the VIs your target will be running.

There are two recommended ways you can customize the NI reference policy; we
will describe each one in detail below.


### Runtime modification of the active SELinux policy

You can modify the active SELinux policy on your real-time target using the
standard SELinux tools that were installed to your target when you installed
packagegroup-ni-selinux.  To do so, you can employ a process like the one
described below.  *Note: These are standard tools, and more comprehensive
documentation on their use is available online through non-NI sources.*

1. Put SELinux into permissive mode and run through the usual operation of your
application.

2. Search `/var/log/messages` for messages tagged with `avc` to view all SELinux
denials or errors encountered.  Go through these messages, and for each denial
that is overly restrictive for your application, add the message to a file
`denials.txt` in the current directory on the target.  An example would be if an
application is supposed to use network resources or access system files but
SELinux denied the access.

3. Run `audit2allow < denials.txt` to generate a new type enforcement policy
fragment based on the messages you selected.  Review the output of the command
to ensure that the policy changes are safe and do not have unintended effects on
your system's security.

4. Run `audit2allow -M policy_all < denials.txt` to generate a new policy
module based on the messages you selected.

5. Run `semodule -i policy_all.pp` to apply the resulting .pp file to the
running SELinux policy.

6. Repeat steps 1-5 as needed.

7. When your policy is complete, use the NI system imaging tools to save a copy
of your image.


### Building an SELinux policy in the OpenEmbedded build environment

NI makes available a set of tools for building packages for our OpenEmbedded
distribution; you can download them from https://github.com/ni/nilrt.  Using
these tools, you can build your own custom version of the refpolicy-standard
.ipk package and install it to your real-time target.  The process looks like
this:

1. Create your own fork of this NI reference policy git repository at a location
that is accessible from your build machine, either on GitHub or on a local
server.

2. Create and check out a branch, make your policy changes as git commits, and
push your branch.

3. Download the NI OpenEmbedded image creation scripts by running
`git clone https://github.com/ni/nilrt`.

4. Run the `nibb.sh` OpenEmbedded image creation script as described in that
repository's README.  This will check out the OpenEmbedded git repositories into
the `./sources` directory.

5. Change to the `sources/meta-nilrt` directory.  From there, edit
`recipes-security/refpolicy/refpolicy-ni.inc`.  Change the first line of the
`SRC_URI` to point to your fork of the NI reference policy and the branch you
created with your policy changes, and change the `SRCREV` to the commit hash of
the top commit in your branch.

6. Following the instructions in the README for the NI OpenEmbedded scripts, set
up your environment and run `bitbake refpolicy-standard`.

7. `scp` the resulting refpolicy-standard .ipk to your real-time target and
install it with `opkg install <filename>`.

8. Reload your real-time target, and the new policy should be activated.

---

Enjoy, and happy hacking!

[ni.com/community/](http://ni.com/community/)
