# windows-fido-bridge

This repository implements [an OpenSSH security key
middleware](https://github.com/openssh/openssh-portable/blob/e9dc9863723e111ae05e353d69df857f0169544a/PROTOCOL.u2f)
that allows you to use [a FIDO/U2F security
key](https://en.wikipedia.org/wiki/Universal_2nd_Factor) (for example, [a
YubiKey](https://www.yubico.com/products/)) to SSH into a remote server from a
machine running Windows 10 with [Windows Subsystem for
Linux](https://docs.microsoft.com/en-us/windows/wsl/about) or [Cygwin](
https://www.cygwin.com/).

## Requirements

At a minimum, you must have the following in order to use this repository:

* A local Linux distribution running inside WSL with OpenSSH 8.3 or newer
  installed.
  ```
  cd ~
  sudo apt update && sudo apt install build-essential zlib1g-dev libssl-dev libpam0g-dev libselinux1-dev
  wget -c https://cloudflare.cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.8p1.tar.gz
  tar -xzf openssh-8.8p1.tar.gz
  rm openssh-8.8p1.tar.gz
  cd openssh-8.8p1
  ./configure --with-md5-passwords --with-pam --with-selinux --with-privsep-path=/var/lib/sshd/ --sysconfdir=/etc/ssh
  sudo make install
  cd ..
  rm -rf openssh-8.8p1
  ```
  Load into a new terminal and `ssh -V` should return `OpenSSH_8.8p1`
* A remote server running OpenSSH 8.2 or newer.
  * The aforementioned API incompatibility does not affect the remote server, so
    it **does not** need OpenSSH 8.3.
* A FIDO/U2F security key that supports Ed25519 or ECDSA.

Cygwin is also supported on a best-effort basis; see the Cygwin section under
Tips below.

## Install

You may want to visit [the
wiki](https://github.com/mgbowen/windows-fido-bridge/wiki/Installing-a-distro-with-OpenSSH-8.3)
that details how to get a Linux distro with a version of OpenSSH that's new
enough to work with windows-fido-bridge.

### From the apt repository

The recommended method of installing windows-fido-bridge is to use its apt
repository at [apt.mgbowen.dev](https://apt.mgbowen.dev). Go to that link and
follow its instructions to set up access to the repository for your operating
system, then run the following:

```
sudo apt install windows-fido-bridge
```

### From source

You can also build this repository from source.

#### C++20::span

You will need a distro that supports c++20::span. Easiest way to do this is to upgrade your WSL2 Ubuntu from 20.04 to 21.04. 
```
sudo apt update
sudo apt upgrade -y
sudo apt purge snapd -y
sudo do-release-upgrade
```

Follow the instructions to edit a config file to `Prompt=normal`, then:

```
sudo do-release-upgrade
```

If prompted to configure postfix for mail, choose leave unchanged.

It will attempt to reboot. It will fail. Simply close the terminal. Open a new PowerShell as Admin and use `wsl --shutdown`. Then open your Ubuntu shell again. Running `lsb_release -a` should now show you as running Ubuntu 21.04.

#### building


```
sudo apt install build-essential cmake g++-mingw-w64-x86-64 git

git clone https://github.com/allar/windows-fido-bridge.git
cd windows-fido-bridge
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j $(nproc)
make test
sudo make install
```

There is a slow down running .exe's off of the wsl2 filesystem apparently, as [this issue](https://github.com/mgbowen/windows-fido-bridge/issues/21) might seem to indicate. Fix for this is to move the windowsfidobridge.exe into /mnt/c/ and them symlink wsl2 to the /mnt/c.

```
sudo mkdir -p /mnt/c/windowsfidobridge/
sudo mv /usr/local/.lib/windowsfidobridge.exe /mnt/c/windowsfidobridge/
sudo ln -s /mnt/c/windowsfidobridge/windowsfidobridge.exe /usr/local/./lib/windowsfidobridge.exe
```
### deb helper

There is also the option of packaging the built binaries into a deb package and
installing that package instead of using `make install`:

```
sudo apt install debhelper

make package
sudo apt install ./windows-fido-bridge_*_*.deb ./windows-fido-bridge-skapi*_*_*.deb
```

Note that if you install the deb package, apt will place the built binaries in
`/usr/lib`, whereas `make install` will place them, by default, in
`/usr/local/lib`. The distinction is important to remember when you set the
`SecurityKeyProvider` option when calling `ssh` or the `SSH_SK_PROVIDER`
environment variable.

#### Compile-time options

You may set the following options when you invoke `cmake`:

* `BUILD_TESTS`: Whether or not to build tests. Defaults to `ON`, set to `OFF`
  to disable.
* `SK_API_VERSION`: The version of the OpenSSH security key API to target. The
  following versions are required to use with their respective OpenSSH versions:
    * `5`: OpenSSH 8.3
    * `7`: OpenSSH 8.4 (default)

## Use

First, you need to generate a key tied to your FIDO/U2F-compliant security key.
To do that, you need to tell OpenSSH what middleware library to use. If you used
the installation instructions above, you can use the following command:

```
SSH_SK_PROVIDER=libwindowsfidobridge.so ssh-keygen -t ed25519-sk -Oapplication=ssh:windows-fido-bridge-verify-required
```

If everything goes well, you should see a Windows dialog pop up asking you to
use your security key. After you confirm and tap your security key, OpenSSH
should then write a public/private key pair to disk. After adding the public key
to your remote server's `.ssh/authorized_keys` file, you can then authenticate
using the following command:

```
ssh -oSecurityKeyProvider=libwindowsfidobridge.so remote-server
```

You should now be logged in to your remote server!

## Tips

### Turn on debug logging

If you're having problems and want more information to help you solve it, or if
you're just curious about what's going on as windows-fido-bridge executes, you
can turn on debug logging by setting the `WINDOWS_FIDO_BRIDGE_DEBUG` environment
variable to any value before executing an OpenSSH executable.

### Force user verification

**Note that this is only possible if _both_ your client and server are running
OpenSSH 8.4 or newer!**

OpenSSH 8.4 added the ability to require user verification in order to log in to
an SSH server via a security key. This means you can require the user to, e.g.
provide a PIN to the security key or place their finger on a security key's
fingerprint reader (if it has one) before being granted access to a remote
server.

When using a security key that requires user verification, OpenSSH will prompt
the user for their PIN and pass that PIN to the security key middleware, which
then passes it to the security key. However, presumably for security reasons,
Microsoft's WebAuthn API does not permit a middleware to prompt for a PIN.
Despite this, OpenSSH will _always_ prompt the user for a PIN before passing
control to a security key middleware, and OpenSSH does not provide the ability
to disable this prompt, which means that you are prompted for a PIN twice: once
from OpenSSH and once from Windows.

To get around this, you can force windows-fido-bridge to create a security key
assertion with user verification even if OpenSSH is configured not to do so,
allowing you to only be prompted for a PIN once by Windows. There are two ways
to enable this behavior:

* When creating an OpenSSH security key-backed SSH key, set the FIDO application
  to `ssh:windows-fido-bridge-verify-required`, like so:
  ```
  SSH_SK_PROVIDER=libwindowsfidobridge.so \
      ssh-keygen -t ecdsa-sk -Oapplication=ssh:windows-fido-bridge-verify-required
  ```
  The key will be created normally; when you use it to log in,
  windows-fido-bridge will ask for a PIN (if that's how your security key
  performs user verification), but OpenSSH will not.
* Set the `WINDOWS_FIDO_BRIDGE_FORCE_USER_VERIFICATION` environment variable to
  any value before logging in to a remote server with `ssh`. You do not need to
  set it before generating the SSH key with `ssh-keygen -t ecdsa-sk`.

Note that it is still possible to create an OpenSSH security key-backed key with
windows-fido-bridge that requires user verification using `ssh-keygen
-Overify-required ...`, and windows-fido-bridge will respect asking for user
verification when logging in with keys that are configured as such.

Finally, you need to enforce that the remote server checks for user verification
before permitting a user to log in with a security key. You can do so by
prepending the public SSH key in your `~/.ssh/authorized_keys` file with
`verify-required`, like so:
```
# ~/.ssh/authorized_keys
verify-required sk-ecdsa-sha2-nistp256@openssh.com AAAA[...]abcdef user@server
```

### Use with ssh-agent

If you want to use a security key-backed SSH key with `ssh-agent`, you should
make sure to either invoke `ssh-add` with the `-S` argument pointing to
`libwindowsfidobridge.so` or set the `SSH_SK_PROVIDER` environment variable
before calling `ssh-add`. Note that you **must** specify the full path to the
library when passing it to `ssh-add` for `ssh-agent` to accept it. For example:

```
ssh-add -S /usr/lib/libwindowsfidobridge.so

# or

SSH_SK_PROVIDER=/usr/lib/libwindowsfidobridge.so ssh-add
```

You may also completely omit the explicit library specification if you place the
`SSH_SK_PROVIDER` environment variable definition in your `.bashrc` or whatever
your shell's equivalent file is.

### Use from Windows

If you want to be able to run `ssh` from a Windows command prompt without first
being in a WSL prompt, you can create a directory somewhere on your Windows
filesystem (for example, `C:\Users\<username>\bin`), add that directory to your
`PATH`, and create a file inside that directory named `ssh.bat` with the
following contents:

```
@wsl ssh %*
```

If the WSL distribution you installed windows-fido-bridge in is not your
default, be sure to pass the `--distribution` argument to `wsl` specifying the
name of the appropriate distribution. Also be sure that you don't have the
Microsoft-distributed OpenSSH client installed or that one may be used instead
of the WSL one.

### Use with Cygwin

windows-fido-bridge supports Cygwin on a best-effort basis; while the primary
execution environment is intended to be WSL, it also happens to be reasonably
easy to compile on Cygwin as well.

To compile in a Cygwin environment, ensure the latest stable versions of the
following packages are installed:

* `cmake`
* `gcc-g++`
* `git`
* `make`

Then, run the standard installation steps as if you were compiling for WSL
(ignore the `apt` commands, of course). The build system will detect that you're
building inside Cygwin and adjust the default options accordingly. The default
build artifact will be a library named `cygwindowsfidobridge.dll`, which is the
file you should specify when telling SSH what SK middleware to use. For example:

```
# Generate a security key-backed SSH key:
SSH_SK_PROVIDER=cygwindowsfidobridge.dll ssh-keygen -t ecdsa-sk

# Use your security key-backed SSH key:
ssh -oSecurityKeyProvider=cygwindowsfidobridge.dll user@remote
```

All other functionality, e.g. changing the middleware's behavior via environment
variables, works the same as it does in WSL.

Note that you cannot use artifacts targeting Cygwin with a non-Cygwin OpenSSH,
and attempting to do so will almost certainly result in a crash when attempting
to pass data back to OpenSSH.

## References

* [Web Authentication: An API for accessing Public Key Credentials, Level
  1](https://www.w3.org/TR/webauthn/)
  * The official W3C WebAuthn specification. Microsoft's API seems to be largely
    based directly on this document.
* [U2F support in OpenSSH
  HEAD](https://marc.info/?l=openssh-unix-dev&m=157259802529972&w=2)
  * Email by Damien Miller announcing the release of OpenSSH's U2F/FIDO support.
