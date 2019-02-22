# Homebrew Asterisk

[![Build Status](https://travis-ci.org/adilinden-oss/homebrew-asterisk.svg?branch=master)](https://travis-ci.org/adilinden-oss/homebrew-asterisk)

Forked from [leedm777/homebrew-asterisk] for my own personal use.

This repo contains the Homebrew formulas I use for my [Asterisk][ast] PBX running on OS X. 

## Installation

    brew tap adilinden/homebrew-asterisk
    brew install asterisk

## Installation options

 * `--with-clang` - Compile with clang instead of gcc.
   * This is a new-ish option in Asterisk, and might be a bit crashy.
 * `--with-dev-mode` - Enable dev mode in Asterisk.
   * Disable optimizations, turns up build warnings, and enables the test
     framework.
 * `--without-optimizations` - disable optimizations.
 * `--with-extra-sounds` - Extra sounds.
   * Download and install extra and core sounds in ulaw, g729 and gsm.

## Configuration

There are two sets of example configuration files.

- `samples`, which contain comments explaining each command option
- `basic-pbx`, which is a functional basic PBX

Because the Asterisk `install samples` will overwite any files present in the `/usr/local/etc/asterisk` directory, I choose to install the samples in more sensible locations.

- `/usr/local/share/doc/asterisk/samples`
- `/usr/local/share/doc/asterisk/basic-pbx`

Place Asterisk configuration files, either your own or files from `samples` or `basic-pbx` in `/usr/local/etc/asterisk`. Detailed configuration docs can be found on the [Asterisk wiki][config-docs].

If you have problems after an upgrade, it may be because of bad path information that ended up in `asterisk.conf`. Check that the directories section looks like:

    [directories](!)
    astetcdir => /usr/local/etc/asterisk
    astmoddir => /usr/local/opt/asterisk/lib/asterisk/modules
    astvarlibdir => /usr/local/var/lib/asterisk
    astdbdir => /usr/local/var/lib/asterisk
    astkeydir => /usr/local/var/lib/asterisk
    astdatadir => /usr/local/var/lib/asterisk
    astagidir => /usr/local/var/lib/asterisk/agi-bin
    astspooldir => /usr/local/var/spool/asterisk
    astrundir => /usr/local/var/run/asterisk
    astlogdir => /usr/local/var/log/asterisk
    astsbindir => /usr/local/opt/asterisk/sbin

## Running asterisk

If you want to just run Asterisk occasionally, just start it up using
`/usr/local/sbin/asterisk -c`. It is recommended to *not* run Asterisk as root.

## Running as a service at user login

To have launchd start asterisk at login:

    mkdir -p ~/Library/LaunchAgents
    ln -sfv /usr/local/opt/asterisk/*.plist ~/Library/LaunchAgents

Then to load asterisk now:

    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.asterisk.plist

To reload asterisk after an upgrade:

    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.asterisk.plist
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.asterisk.plist

To connect to Asterisk running as a service:

    /usr/local/sbin/asterisk -r

To restart asterisk after a `core stop now`:

    launchctl start homebrew.mxcl.asterisk

## Running as a system service

It is recommended to *not* run Asterisk as root.  A dedicated system user for asterisk can be created using the following procedure.

### Create the asterisk user

Use the OS X dscl tool to create a new user account for asterisk.  Start by finding a uid and gid that is not in use.  This little script can be pasted into a terminal window to find the next uid and guid combo above 301 that is not in use.

```bash
    for (( uid = 301; uid<500; uid++ )) ; do \
        if ! id -u $uid &>/dev/null; then \
            if ! dscl . -ls Groups gid | grep -q [^0-9]$uid\$ ; then \
                 echo Found: $uid; \
                 export this_id=$uid; \
                 break; \
            fi; \
        fi; \
    done;
```

Now create the asterisk user account.  If the above snippet was run the $this_id variable will already contain the uid and gid to use.

```bash
    sudo dscl . -create /Groups/_asterisk
    sudo dscl . -create /Groups/_asterisk Password \*
    sudo dscl . -create /Groups/_asterisk PrimaryGroupID $this_id
    sudo dscl . -create /Groups/_asterisk RealName "Asterisk Daemon"
    sudo dscl . -create /Groups/_asterisk RecordName _asterisk asterisk

    sudo dscl . -create /Users/_asterisk
    sudo dscl . -create /Users/_asterisk NFSHomeDirectory /usr/local/var/lib/asterisk
    sudo dscl . -create /Users/_asterisk Password \*
    sudo dscl . -create /Users/_asterisk PrimaryGroupID $this_id
    sudo dscl . -create /Users/_asterisk RealName "Asterisk Daemon"
    sudo dscl . -create /Users/_asterisk RecordName _asterisk asterisk
    sudo dscl . -create /Users/_asterisk UniqueID $this_id
    sudo dscl . -create /Users/_asterisk UserShell /bin/bash
    sudo dscl . -create /Users/_asterisk IsHidden 1

    sudo dscl . -delete /Users/_asterisk AuthenticationAuthority
    sudo dscl . -delete /Users/_asterisk PasswordPolicyOptions
```

Fix directory permissions of directories that the new asterisk user requires write access to.

```bash
    sudo chown -R asterisk:asterisk /usr/local/var/lib/asterisk
    sudo chown -R asterisk:asterisk /usr/local/var/log/asterisk
    sudo chown -R asterisk:asterisk /usr/local/var/run/asterisk
    sudo chown -R asterisk:asterisk /usr/local/var/spool/asterisk
```

### Launch the asterisk service

Modify `/usr/local/opt/asterisk/homebrew.mxcl.asterisk.plist` and add keys to specify the user and group we generated earlier.

```xml
    <key>UserName</key>
    <string>asterisk</string>
    <key>GroupName</key>
    <string>asterisk</string>
```

Install the modified `/usr/local/opt/asterisk/homebrew.mxcl.asterisk.plist` in `/Library/LaunchDaemons`.

    sudo cp /usr/local/opt/asterisk/homebrew.mxcl.asterisk.plist /Library/LaunchDaemons/

Then to load asterisk now:

    sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.asterisk.plist

To reload asterisk after an upgrade:

    sudo launchctl unload /Library/LaunchDaemons/homebrew.mxcl.asterisk.plist
    sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.asterisk.plist

To connect to Asterisk running as a service:

    sudo -u asterisk /usr/local/sbin/asterisk -r

To restart asterisk after a `core stop now`:

    sudo launchctl start homebrew.mxcl.asterisk

## Uninstall

To uninstall Asterisk, run `brew rm asterisk`. To get rid of all local state and configuration data:

    $ rm -rf /usr/local/etc/asterisk /usr/local/var/lib/asterisk \
        /usr/local/var/log/asterisk /usr/local/var/run/asterisk \
        /usr/local/var/spool/asterisk

## Upgrading from older versions

I used to have notes in the README for creating an `asterisk` user for running
Asterisk. This is not very homebrew-y, so I dropped it.

I also had a custom plist that I recommended instead of homebrew's built-in
plist feature. If you had followed those instructions, you may need to remove
`/Library/LaunchDaemons/org.asterisk.asterisk.plist` before installing
[the new plist above](#Running as a service).

 [ast]: http://asterisk.org/
 [config-docs]: https://wiki.asterisk.org/wiki/x/cYXAAQ
 [leedm777/homebrew-asterisk]: https://github.com/leedm777/homebrew-asterisk
 
