---
title: "Sign GitHub Commits"
date: 2022-09-18
published: false
---

For sure you've seen that some commits are marked as `Verified` on GitHub but the most are not. GitHub automatically marks commits as `Verified` when they were made using the GitHub web interface. `Verified` means that the commits are signed by their author so you can be sure that they were made by that author. Although it can look like not a big trouble it can be a good idea to sign your own commits.

You can use `gpg` or `ssh` keys for signing[^Read more in the "[Verify commit signatures][github-docs]" section of GitHub Docs.].

## Set up GPG

Install GPG. The easiest way is to use Homebrew:

```zsh
brew install gpg
```

Enable the GPG agent to avoid having to type the secret key’s password every time. To do that, add `use-agent` key to `~/.gnupg/gpg.conf`:

```zsh
mkdir -m 700 ~/.gnupg

cat << EOF >> ~/.gnupg/gpg.conf
# Tell GPG to use the gpg-agent
use-agent
EOF
```

Set the `GPG_TTY` variable required by the `gpg-agent`[^`gpg-agent` environment requirements: "[Invoking GPG-AGENT][invoking-gpg-agent]"] in `~/.zshrc`:

```zsh
cat << EOF >> ~/.zshrc

# Set the GPG_TTY variable required by the gpg-agent
export GPG_TTY=$(tty)
EOF
```

Restart your Terminal or run:

```zsh
source ~/.zshrc
```

## Generate your GPG key pair

Generate a GPG key pair[^"[Generating a new GPG key][generating-a-new-gpg-key]" on GitHub Docs]:

```zsh
gpg --full-generate-key
```

It'll ask a few questions about the key. The suggested default options are good so you can use them:

```zsh
% gpg --full-generate-key
gpg (GnuPG) 2.3.7; Copyright (C) 2021 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

gpg: keybox '/Users/aleos/.gnupg/pubring.kbx' created
Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 
Key does not expire at all
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: Alexander Ostrovsky
Email address: thealeos@gmail.com
Comment: GitHub key
You selected this USER-ID:
    "Alexander Ostrovsky <thealeos@gmail.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
You need a Passphrase to protect your secret key.
```

## Add the GPG key to GitHub

List the GPG keys:

```zsh
gpg --list-secret-keys --keyid-format=long
```

Copy the long form of the GPG key ID. In this example, the GPG key ID is `6886EEFB3208FB51`:

```zsh
% gpg --list-secret-keys --keyid-format=long
/Users/aleos/.gnupg/pubring.kbx
-------------------------------
sec   ed25519/6886EEFB3208FB51 2022-09-17 [SC]
      E50EB4FD3B38C85B0954EC626886EEFB3208FB51
uid                 [ultimate] Alexander Ostrovsky (GitHub key) <thealeos@gmail.com>
ssb   cv25519/45F1C3043B776FD2 2022-09-17 [E]
```

Copy your GPG key:

```zsh
gpg --armor --export 6886EEFB3208FB51 | pbcopy
```

Add this key to "GPG keys" section in your GitHub profile settings[^"[Adding a GPG key to your GitHub account][add-gpg-github]" on GitHub Docs].

![GPG keys / Add new](/docs/assets/sign-github-commits/github-gpg-keys-add-new.png)

## Sign Commits

Add signing key to the `git config`:

```zsh
git config --global user.signingkey 6886EEFB3208FB51
```

Tell `git` to sign all commits:

```zsh
git config --global commit.gpgsign true
```

Unfortunately UI tools like Xcode unable to request the passphrase. To solve it install `pinentry-mac`:

```zsh
brew install pinentry-mac
```

And tell `gnupg` the path to it using the key `pinentry-program` in the `gpg-agent.conf`:

```zsh
echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf
```

Then reload the `gpg-agent` to apply the changes:

```zshrc
gpg-connect-agent reloadagent /bye
```

or restart it with:

```zsh
gpgconf --kill gpg-agent
gpgconf --launch gpg-agent
```

Configure Git to use gpg

```zsh
git config --global gpg.program $(which gpg)
```

## (Optional) Backup your private key

To backup your private key do the following:

```zsh
gpg --export-secret-keys --armor --output pgp-private-key.asc thealeos@gmail.com
```

You should store the backup in a really safe place. To import the backup when you need it:

```zsh
gpg --import pgp-private-key.asc
```

You might want to backup your revocation certificates located in `~/.gnupg/openpgp-revocs.d/`. Read more about using GnuPG in [archlinux wiki][archlinux-gnupg].



==Check that pinentry-mac saved passphrase in your keychain: security find-generic-password -s 'GnuPG'==



10.6 Put a passphrase into the cache
The gpg-preset-passphrase is a utility to seed the internal cache of a running gpg-agent with passphrases. It is mainly useful for unattended machines, where the usual pinentry tool may not be used and the passphrases for the to be used keys are given at machine startup.



[github-docs]: https://docs.github.com/en/authentication/managing-commit-signature-verification "Verify commit signatures"
[invoking-gpg-agent]: https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html "Invoking GPG-AGENT"
[generating-a-new-gpg-key]: https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key "Generating a new GPG key"
[add-gpg-github]: https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account "Adding a GPG key to your GitHub account"
[archlinux-gnupg]: https://wiki.archlinux.org/title/GnuPG "GnuPG"