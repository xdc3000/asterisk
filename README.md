# The Asterisk(R) Open Source PBX

> ## Fork note: chan_mobile mSBC wideband audio
>
> This is a fork of Asterisk that adds **mSBC (HFP wideband, 16 kHz) audio
> support to `chan_mobile`**. Stock `chan_mobile` only does narrowband CVSD
> (8 kHz); this fork negotiates and runs the mSBC codec over HFP, roughly
> doubling call-audio bandwidth when bridging a Bluetooth phone's cellular
> calls into Asterisk.
>
> ### What changed (vs upstream)
>
> - **`addons/chan_mobile.c`**
>   - HFP codec negotiation: advertise codec support in BRSF, send
>     `AT+BAC=1,2`, and handle the AG's `+BCS` to select mSBC (codec 2) or
>     fall back to CVSD (codec 1).
>   - SCO accept side: `BT_DEFER_SETUP` on the listener plus `BT_VOICE`
>     transparent air mode for mSBC links.
>   - libsbc mSBC encode/decode with H2 framing; the channel runs natively at
>     `slin16` and the CVSD path resamples to/from 16 kHz so the channel format
>     is constant for the call.
>   - TX is paced to the eSCO air clock: encoded bytes are queued and written
>     to the SCO socket in fixed air-frame-sized packets (size learned from the
>     RX packet length), flushed non-blocking so the kernel USB-isochronous
>     clock sets the rate instead of the bursty PBX write thread.
>   - RX wall-clock drift compensation with silence-aware drop/repeat, so the
>     unavoidable SCO-vs-system clock correction lands in speech gaps.
> - **`addons/Makefile`** — links `chan_mobile` against `libsbc` (`-lsbc`).
>
> ### Requirements
>
> - `libsbc` / `libsbc-dev` (Debian/Ubuntu).
> - A USB Bluetooth adapter whose controller supports **transparent (mSBC)
>   eSCO** — check `hciconfig <hci> features` for `<transparent SCO>`.
> - A phone that performs **HFP codec negotiation** (sends `+BCS`). This is the
>   real compatibility gate: a phone that never sends `+BCS` will only ever get
>   CVSD (or fail to set up SCO), no matter the adapter. (For example, a Redmi
>   9A in testing never negotiated mSBC; a Huawei Mate60 Pro does.)
>
> ### Tested configuration
>
> | Component | Detail |
> | --- | --- |
> | Host / OS | Mac mini (x86_64), Ubuntu 24.04 |
> | Asterisk | 20.6 (addons / `chan_mobile`) |
> | USB Bluetooth adapter | **CSR8510 A10** dongle — USB ID `0a12:0001` (Cambridge Silicon Radio); HCI SCO MTU 64 |
> | Phone | **Huawei Mate60 Pro** (HFP-AG, negotiates mSBC via `+BCS`) |
> | Result | Bidirectional 16 kHz wideband call, SIP leg at G.722 |
>
> ### Build
>
> ```sh
> sudo apt-get install libsbc-dev            # Debian/Ubuntu
> ./configure
> make
> sudo make install
> ```
>
> `chan_mobile` selects mSBC automatically when the phone supports it and falls
> back to CVSD otherwise. Configure the device in `chan_mobile.conf` as usual
> (the adapter `address` and the phone `address` / HFP-AG RFCOMM `port`); no
> mSBC-specific configuration is required.
>
> This fork is provided as-is and is not affiliated with the Asterisk project.

```
By Mark Spencer <markster@digium.com> and the Asterisk.org developer community.
Copyright (C) 2001-2025 Sangoma Technologies Corporation and other copyright holders.
```

## SECURITY

It is imperative that you read and fully understand the contents of
the security information document before you attempt to configure and run
an Asterisk server.

See [Important Security Considerations](https://docs.asterisk.org/Deployment/Important-Security-Considerations) for more information.

## WHAT IS ASTERISK ?

Asterisk is an Open Source PBX and telephony toolkit.  It is, in a
sense, middleware between Internet and telephony channels on the bottom,
and Internet and telephony applications at the top.  However, Asterisk supports
more telephony interfaces than just Internet telephony.  Asterisk also has a
vast amount of support for traditional PSTN telephony, as well.

For more information on the project itself, please visit the [Asterisk
Home Page](https://www.asterisk.org) and the official
[Asterisk Documentation](https://docs.asterisk.org).

## SUPPORTED OPERATING SYSTEMS

### Linux

The Asterisk Open Source PBX is developed and tested primarily on the
GNU/Linux operating system, and is supported on every major GNU/Linux
distribution.

### Others

Asterisk has also been 'ported' and reportedly runs properly on other
operating systems as well, Apple's Mac OS X, and the BSD variants.

## GETTING STARTED

Most users are using VoIP/SIP exclusively these days but if you need to
interface to TDM or analog services or devices, be sure you've got supported
hardware.

Supported telephony hardware includes:
* All Analog and Digital Interface cards from Sangoma
* Any full duplex sound card supported by PortAudio
* The Xorcom Astribank channel bank

### UPGRADING FROM AN EARLIER VERSION

If you are updating from a previous version of Asterisk, make sure you
read the Change Logs.

<!-- CHANGELOGS (the URL will change based on the location of this README) -->
[Change Logs](https://downloads.asterisk.org/pub/telephony/asterisk)
<!-- END-CHANGELOGS -->

### NEW INSTALLATIONS

Ensure that your system contains a compatible compiler and development
libraries.  Asterisk requires either the GNU Compiler Collection (GCC) version
4.1 or higher, or a compiler that supports the C99 specification and some of
the gcc language extensions.  In addition, your system needs to have the C
library headers available, and the headers and libraries for ncurses.

There are many modules that have additional dependencies.  To see what
libraries are being looked for, see `./configure --help`, or run
`make menuselect` to view the dependencies for specific modules.

On many distributions, these dependencies are installed by packages with names
like 'glibc-devel', 'ncurses-devel', 'openssl-devel' and 'zlib-devel'
or similar.  The `contrib/scripts/install_prereq` script can be used to install
the dependencies for most Debian and Redhat based Linux distributions.
The script also handles SUSE, Arch, Gentoo, FreeBSD, NetBSD and OpenBSD but
those distributions mightnoit have complete support or they might be out of date.

So, let's proceed:

1. Read the documentation.<br>
The [Asterisk Documentation](https://docs.asterisk.org) website has full
information for building, installing, configuring and running Asterisk.

2. Run `./configure`<br>
Execute the configure script to guess values for system-dependent
variables used during compilation. If the script indicates that some required
components are missing, you can run `./contrib/scripts/install_prereq install`
to install the necessary components. Note that this will install all dependencies
for every functionality of Asterisk. After running the script, you will need
to rerun `./configure`.

3. Run `make menuselect`<br>
This is needed if you want to select the modules that will be compiled and to
check dependencies for various optional modules.

4. Run `make`<br>
Assuming the build completes successfully:

5. Run `make install`<br>
If this is your first time working with Asterisk, you may wish to install
the sample PBX, with demonstration extensions, etc.  If so, run:

6. Run `make samples`<br>
Doing so will overwrite any existing configuration files you have installed.

7. Finally, you can launch Asterisk in the foreground mode (not a daemon) with
`asterisk -vvvc`<br>
You'll see a bunch of verbose messages fly by your screen as Asterisk
initializes (that's the "very very verbose" mode).  When it's ready, if
you specified the "c" then you'll get a command line console, that looks
like this:<br>
`*CLI>`<br>
You can type `core show help` at any time to get help with the system.  For help
with a specific command, type `core show help <command>`.

`man asterisk` at the Unix/Linux command prompt will give you detailed
information on how to start and stop Asterisk, as well as all the command
line options for starting Asterisk.

### ABOUT CONFIGURATION FILES

All Asterisk configuration files share a common format.  Comments are
delimited by `;` (since `#` of course, being a DTMF digit, may occur in
many places).  A configuration file is divided into sections whose names
appear in `[]`'s.  Each section typically contains statements in the form
`variable = value` although you may see `variable => value` in older samples.

### SPECIAL NOTE ON TIME

Those using SIP phones should be aware that Asterisk is sensitive to
large jumps in time.  Manually changing the system time using date(1)
(or other similar commands) may cause SIP registrations and other
internal processes to fail.  For this reason, you should always use
a time synchronization package to keep your system time accurate.
All OS/distributions make one or more of the following packages
available:

* ntpd/ntpsec
* chronyd
* systemd-timesyncd

Be sure to install and configure one (and only one) of them.

### FILE DESCRIPTORS

Depending on the size of your system and your configuration,
Asterisk can consume a large number of file descriptors.  In UNIX,
file descriptors are used for more than just files on disk.  File
descriptors are also used for handling network communication
(e.g. SIP, IAX2, or H.323 calls) and hardware access (e.g. analog and
digital trunk hardware).  Asterisk accesses many on-disk files for
everything from configuration information to voicemail storage.

Most systems limit the number of file descriptors that Asterisk can
have open at one time.  This can limit the number of simultaneous
calls that your system can handle.  For example, if the limit is set
at 1024 (a common default value) Asterisk can handle approximately 150
SIP calls simultaneously.  To change the number of file descriptors
follow the instructions for your system below:

#### PAM-BASED LINUX SYSTEM

If your system uses PAM (Pluggable Authentication Modules) edit
`/etc/security/limits.conf`.  Add these lines to the bottom of the file:

```text
root            soft    nofile          4096
root            hard    nofile          8196
asterisk        soft    nofile          4096
asterisk        hard    nofile          8196
```

(adjust the numbers to taste).  You may need to reboot the system for
these changes to take effect.

#### GENERIC UNIX SYSTEM

If there are no instructions specifically adapted to your system
above you can try adding the command `ulimit -n 8192` to the script
that starts Asterisk.

## MORE INFORMATION

Visit the [Asterisk Documentation](https://docs.asterisk.org) website
for more documentation on various features and please read all the
configuration samples that include documentation on the configuration options.

Finally, you may wish to join the
[Asterisk Community Forums](https://community.asterisk.org)


Welcome to the growing worldwide community of Asterisk users!

```
        Mark Spencer, and the Asterisk.org development community
```

---

Asterisk is a trademark of Sangoma Technologies Corporation

\[[Sangoma](https://www.sangoma.com/)\] 
\[[Home Page](https://www.asterisk.org)\] 
\[[Support](https://www.asterisk.org/support)\] 
\[[Documentation](https://docs.asterisk.org)\] 
\[[Community Forums](https://community.asterisk.org)\] 
\[[Release Notes](https://github.com/asterisk/asterisk/releases)\] 
\[[Security](https://docs.asterisk.org/Deployment/Important-Security-Considerations/)\] 
\[[Mailing List Archive](https://lists.digium.com)\] 

