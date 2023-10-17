---
layout: post
title: Joel's Opinionated Guide to GPG and SSH on the YubiKey
---

A long time ago I got a (YubiKey 5C Nano)[https://support.yubico.com/hc/en-us/articles/360013724699-YubiKey-5C-Nano], and didn't use it. All the noise
around Passkeys made me decide to get it out, which was a waste of time because
Linux support for Passkeys doesn't yet exist. But they can be used for FIDO and
as a GPG smartcard.

## GPG Smartcard

I mostly followed the kernel.org maintainer documentation:

 * [https://docs.kernel.org/process/maintainer-pgp-guide.html]
 * [https://github.com/lfit/itpol/blob/master/protecting-code-integrity.md]

### Pins

The important information that kept on tripping me up:

 * The default pin is 123456
 * The default master pin is 12345678.

Both of these should be changed. They can be any ascii, up to 127 characters.
The admin pin is used when adding keys to the device, and the other pin is used
when using the device to sign something. The UI for when you get the pin wrong
is horrible; I had to destructively reset the device a few times before I got
it right.

### Subkey tips

Once you've set that up, the instructions to generate subkeys from your GPG key
make sense. I've only ever used a single Signing/Certify key with my GPG key,
so this was an adventure in creating and using subkeys.

An important detail is that adding the keys to your smartcard (Yubikey) is a
destructive operation: the private keys no longer exist on your laptop. This is
why the instructions suggest making a backup of the `~/.gnupg` directory before
moving them to the Yubikey. You can test this works by setting `GNUPGHOME` and
printing out your key.

```
$ GNUPGHOME=/media/joel/key-backup/gnupg-dir gpg --list-secret-keys
-----------------------------
sec   rsa4096 2014-03-04 [SC]
      FA71CC02DF4F0810C7EB7C016B7659078147709E
uid           [ultimate] Joel Stanley <joel@kernel.org>
uid           [ultimate] Joel Mark Stanley <joel@jms.id.au>
ssb   rsa4096 2014-03-04 [E]
ssb   ed25519 2023-10-15 [S]
ssb   ed25519 2023-10-16 [A]
```

The kernel guide recommends using a LUKS encrpyted USB key or two for backups.
LUKS encrypting a USB key is trivial with the Gnome Disks application.

Similarly mounting it is easy: Gnome will prompt you for the passphrase and it
presents as normal. I'm not sure what I'll do if/when I move back to using
Sway.


### Other Notes

I learnt that the expiry date can be updated, and if your key has "expired" you
can re-publish the same public key with an updated expiry date:

```
gpg --quick-set-expire [fpr] 1y
```

If you're a kernel maintainer, you can also add your kernel.org email address
as an alias, allowing it to be discovered with
[WKD](https://wiki.gnupg.org/WKD) (Web Key Directory).

 [https://korg.docs.kernel.org/mail.html#adding-a-kernel-org-uid-to-your-pgp-key]

```
gpg --quick-add-uid [keyid] 'Firstname Lastname <username@kernel.org>'
```

### YubiKey Settings

By default you don't need to touch the device when using one of your keys,
which takes away all of the fun of using a Yubikey IMO. You can change that
using ykman:

 [https://docs.yubico.com/software/yubikey/tools/ykman/OpenPGP_Commands.html]

This requires a touch when using the encryption keyslot, but remembers the touch
for 15 seconds (cached), and ensures the setting can't be changed without removing the
device from the USB slot (fixed):

```
ykman openpgp keys set-touch enc cached-fixed
```

This also lets you change the number of times you type in the pin before it
locks you out. I upped it from 3 to 5:

```
ykman openpgp access set-retries 5 5 5 -f 
```

```
$ ykman openpgp info
OpenPGP version: 3.4
Application version: 5.2.7

PIN tries remaining: 5
Reset code tries remaining: 0
Admin PIN tries remaining: 5

Touch policies
Signature key           Cached (fixed)
Encryption key          Cached (fixed)
Authentication key      Cached (fixed)
Attestation key         Off
```

## FIDO Security Key

### Browser stuff

Gitlab and Github both let me add device as a security key. This first requires
setting a pin, which is different to the pin used for the smartcard.

### SSH login

You can use your GPG key with gnupg as your ssh-agent to do ssh auth.  This is
awkward, because it means setting SSH_SOCK_AUTH for your system to the gnupg
socket, meaning it's an all or nothing setup.

Instead I chose to use the FIDO2 key stuff, which lets us generate ssh keys
that exist on the Yubikey, with some metadata on your laptop that points to the
key and lets you use the usual ssh agent. 

 [https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html]

```
ssh-keygen -t ed25519-sk -O resident -O verify-required
```

Then do the usual dance of ssh-copy-id or sending the contents of
`~/.ssh/id_ed25519_sk.pub` to your sysadmin. This key can also be used for
Github and Gitlab.

#### Discoverable vs Non-discoverable

One of the challenges in using this stuff is the documentation is verbose, and
there are many options, to choose between, leaving you feeling like you need to
become an expert in order to make a decision. GPG smartcard vs Discoverable
(Resident) Keys vs Non-discoverable Keys, etc.

I'm using discoverable resident keys, because the documentation made them sound
more convenient. It means the key (and the pin for your key) is all you
require to login; in theory you could take it to a new workstation and get
access to your machine. The non-discoverable key type means you need some
metadata (the public key for your Yubikey?) in order to use it, which seems
like more things to go wrong.

## Feedback and corrections

Please get in touch if you disagree or notice a mistake.
