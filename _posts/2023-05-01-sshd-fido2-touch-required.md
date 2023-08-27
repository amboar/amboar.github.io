---
title: touch-required FIDO2 Authentication with sshd and a Yubikey on Fedora 38
author: Andrew
---

In [OpenBMC Development on an Apple M1 MacBook Pro][workflow] I outlined setting up a
Fedora VM that met a bunch of my workflow requirements for developing OpenBMC.
Mentioned briefly in there was that I was using bridged rather than shared
networking[^1]. A separate concern is that I use SSH to access the VM rather
than doing work through the console. These strategies combined mean that `sshd`
is exposed on my local network, reachable by anyone that happens to have access
to it. Because I'm lazy the network contains printers and other devices whose
firmware hygiene generally causes infosec side-eye. Leaving `sshd` exposed to
password-based authentication attempts didn't evoke feelings of comfort.

[workflow]: /notes/2023/03/27/openbmc-development-on-an-apple-m1-macbook-pro.html

Given it's generally hard for printers to press buttons on machines across the
room I figured a better approach was for `sshd` to require demonstration of
physical presence with a Yubikey.

Yubico [already documents the process of setting up OpenSSH for FIDO2
authentication][yubico-fido2-ssh], however there are some complications, hence
this post.

[yubico-fido2-ssh]: https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html

The first complication is that MacOS doesn't enable support for FIDO2, so the
first action was to [install homebrew][homebrew] and then `brew install
openssh`[^2].  A further problem of note was that despite:

[homebrew]: https://brew.sh/

1. The deployment of the `openssh` keg,
2. The presence of `/opt/homebrew/bin` as the first element of `$PATH`, and
3. Invoking `hash -r`

`command -v ssh-keygen` still pointed to the MacOS bundled binary at
`/usr/bin/ssh-keygen`. I ended up having to quit the shell and restart
`terminal.app` before the shell picked up `/opt/homebrew/bin/ssh-keygen`. I'm
not sure what went wrong there but it seems to be solved now, so 🤷.

Anyway, we now have a usable SSH build for the purpose of FIDO2 auth, so
onwards.

The next choices are discoverable vs non-discoverable keys and the stength of
the autentication mode. I'm just trying to stop my printer from logging into my
VM, so several simplifying choices can be made here:

1. Use of discoverable keys
2. Physical presence is enough protection

Discoverable keys means we at most need the physical Yubikey and nothing more.
Requiring at most physical presence means that OpenSSH's `touch-required` is
all we need rather than `verify-required` in the `sshd` configuration.

With that resolved we can work our way through Yubico's instructions. The next
step was to generate the key[^3]. Simply translating `verify-required` to
`touch-required` in the `ssh-keygen` invocation ends badly[^4], but it turns out
we can just remove specifying the option altogether and get the desired outcome:

```
% ssh-keygen -vv -t ecdsa-sk -O resident -O application=ssh:vm-linux
Generating public/private ecdsa-sk key pair.
You may need to touch your authenticator to authorize key generation.
debug1: start_helper: starting /opt/homebrew/Cellar/openssh/9.3p1/libexec/ssh-sk-helper
debug1: sshsk_enroll: provider "internal", device "(null)", application "ssh:vm-linux", userid "(null)", flags 0x21, challenge len 0
debug1: sshsk_enroll: using random challenge
debug1: sk_probe: 1 device(s) detected
debug1: sk_probe: selecting sk by touch
debug1: ssh_sk_enroll: using device ioreg://4295408457
debug1: check_sk_options: option uv is unknown
debug1: key_lookup: fido_dev_get_assert: FIDO_ERR_NO_CREDENTIALS
debug1: ssh_sk_enroll: attestation cert len=705
debug1: ssh_sk_enroll: authdata len=164
debug1: main: reply len 1153
Enter file in which to save the key (/Users/andrew/.ssh/id_ecdsa_sk):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /Users/andrew/.ssh/id_ecdsa_sk
Your public key has been saved in /Users/andrew/.ssh/id_ecdsa_sk.pub
...
```

With that we copy the new key ID into the VM:

```
% ssh-copy-id -i ~/.ssh/id_ecdsa_sk.pub $VM
```

After that we need to tweak `authorized_keys` to specify `touch-required` as
outlined in the Yubico documentation.

With that done we need to configure `sshd` in the VM to our requirements:

```
# cat <<EOF > /etc/ssh/sshd_config.d/70-fido2-auth.conf
PubkeyAuthOptions touch-required
PasswordAuthentication no
EOF
# systemctl restart sshd
```

From there attempting to log in hit issues with SELinux in the VM, likely as a
result of my kinda strange setup with `$HOME`:

```
May 01 10:15:30 fedora setroubleshoot[1730]: SELinux is preventing sshd from getattr access on the file /mnt/host/andrew/home/andrew/.ssh/authorized_keys. For complete SELinux messages run: sealert -l deb308b7-b71f-4195-8b1a-171a50a0baff
```

After consulting with [man 8 sshd_selinux][systutorials-man-8-sshd_selinux] the
following invocations allow appropriate accesses for `sshd`:

[systutorials-man-8-sshd_selinux]: https://www.systutorials.com/docs/linux/man/8-sshd_selinux/

```
# semanage fcontext -a -t ssh_home_t /mnt/host/andrew/home/andrew/.ssh/authorized_keys
# restorecon -v /mnt/host/andrew/home/andrew/.ssh/authorized_keys
```

I have no idea what I'm doing with SELinux, so if there are better ways to do
that then let me know.

Anyway, with that resolved, success!

```
andrew@Andrews-MacBook-Pro ~ % ssh $VM
Confirm user presence for key ECDSA-SK SHA256:lbtP2AvHJR65p2/u7zKJ+L9CzBTE/pA3YLUxcBuMZxA
User presence confirmed

Last login: Mon May  1 10:30:43 2023 from 192.168.68.102
1 10:33:28 andrew@fedora:~$
```

[^1]: I wanted bridged networking so I didn't have to fiddle with forwarding
    configurations and such, but as it turned out I couldn't get DNS functioning
    when I did try shared networking as an experiment 🤔. A problem for another
    day.

[^2]: In a past effort I'd already installed homebrew on the M1 but then removed
    it after failing to achieve a functioning workflow. It turns out that
    removing homebrew didn't remove the installed applications or configuration.
    While poking around I discovered `brew deps --installed --graph`, which
    combined with `brew uninstall ...` and `brew autoremove` allowed me to
    quickly clean up a lot of cruft.

[^3]: Initially I tried generating a `verify-required`-style key but that ended
	in the following output. In the end I decided I didn't want
    `verify-required` anyway, so it didn't matter:

    ```
	andrew@Andrews-MacBook-Pro ~ % ssh-keygen -vv -t ecdsa-sk -O resident -O application=ssh:vm-linux -O verify-required
	Generating public/private ecdsa-sk key pair.
	You may need to touch your authenticator to authorize key generation.
	debug1: start_helper: starting /opt/homebrew/Cellar/openssh/9.3p1/libexec/ssh-sk-helper 
	debug1: sshsk_enroll: provider "internal", device "(null)", application "ssh:vm-linux", userid "(null)", flags 0x25, challenge len 0
	debug1: sshsk_enroll: using random challenge
	debug1: sk_probe: 1 device(s) detected
	debug1: sk_probe: selecting sk by touch
	debug1: ssh_sk_enroll: using device ioreg://4295408457
	debug1: check_sk_options: option uv is unknown
	debug1: key_lookup: fido_dev_get_assert: FIDO_ERR_NO_CREDENTIALS
	debug1: ssh_sk_enroll: fido_dev_make_cred: FIDO_ERR_PIN_NOT_SET
	debug1: sshsk_enroll: provider "internal" failure -1
	debug1: ssh-sk-helper: Enrollment failed: invalid format
	debug1: main: reply len 8
	debug1: client_converse: helper returned error -4
	Key enrollment failed: invalid format
	```

[^4]: At least the output is user-friendly this time:

    ```
    andrew@Andrews-MacBook-Pro ~ % ssh-keygen -vv -t ecdsa-sk -O resident -O application=ssh:vm-linux -O touch-required
    Generating public/private ecdsa-sk key pair.
    Option "touch-required" is unsupported for FIDO authenticator enrollment
    ```
