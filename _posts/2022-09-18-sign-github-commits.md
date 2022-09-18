---
title: "Sign GitHub Commits"
date: 2022-09-18
---

For sure you've seen that some commits are marked as `Verified` on GitHub but the most are not. GitHub automatically marks commits as `Verified` when they were made using the GitHub web interface. It means that the commits were signed by their author so you can be sure that they were made by that author. Although it can look like not a big trouble it can be a good idea to sign your own commits.

You can use `gpg` or `ssh` keys for signing.[^verify-commit-signatures]

## Set up GPG

Install the [`gpg`][gnupg] utility. The easiest way is to use `Homebrew`:

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

Set the `GPG_TTY` variable required[^gpg-agent-env-requirements] by the `gpg-agent` in `~/.zshrc`:

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

Generate a GPG key pair[^gen-new-gpg-key]:

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

## Add your GPG key to GitHub

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

Add this key to "GPG keys" section in your GitHub profile settings.[^add-gpg-github]

![GPG keys / Add new](/docs/assets/sign-github-commits/github-gpg-keys-add-new.png)

## Sign your commits

### Using command line

Add signing key to the `git config`:

```zsh
git config --global user.signingkey 6886EEFB3208FB51
```

Tell `git` to sign all your commits:

```zsh
git config --global commit.gpgsign true
```

From now all your commits will be signed. It can be overkill for you. It is possible to set `commit.gpgsign` on per-repository basis or only for certain commits using `-S` option with `git commit` command.

`gpg` will ask you to enter the passphrase to unlock the OpenPGP secret key when it's called for the first time:

![Enter the passphrase](/docs/assets/sign-github-commits/passphrase-terminal.png)

After making a signed commit you can check that the commit was signed:

```zsh
git log --show-signature -1
```

With a result like this:

```zsh
% git log --show-signature -1
commit e20d871441001833f7bc024a9140236d7cac71e9 (HEAD -> sign-github-commits, origin/sign-github-commits)
gpg: Signature made Sun Sep 18 12:08:41 2022 MSK
gpg:                using EDDSA key E50EB4FD3B38C85B0954EC626886EEFB3208FB51
gpg: Good signature from "Alexander Ostrovsky (GitHub key) <thealeos@gmail.com>" [ultimate]
Author: Alexander Ostrovsky <thealeos@gmail.com>
Date:   Sun Sep 18 12:08:40 2022 +0300

    Add a new article "Sign GitHub Commits"
```

### Using apps

Tell `git` a path to the `gpg` to help apps to find it:

```zsh
git config --global gpg.program $(which gpg)
```

Unfortunately not all apps can request the passphrase because `pinentry` (`gpg-agent` uses to unlock the secret key) is a command line utility. For example `Atom` can but `Xcode` can't. To solve it install `pinentry-mac`:

```zsh
brew install pinentry-mac
```

And tell `gnupg` the path to it in the `gpg-agent.conf`:

```zsh
echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf
```

Then reload the `gpg-agent` to apply the changes:

```zshrc
gpg-connect-agent reloadagent /bye
```

From now if `gpg-agent` hasn't cached passphrase yet it will ask you by using a `pinentry-mac` utility:

![Pinentry Mac](/docs/assets/sign-github-commits/pinentry-mac.png)

Unfortunately although `pinentry-mac` has an option to save credentials to your `Keychain` I was unable to make it work. You can check whether `pinentry-mac` saved passphrase in your keychain:

```zsh
security find-generic-password -s 'GnuPG'
```

## 🎁 Bonus

You can find lots of additional info about using GnuPG in [archlinux wiki][archlinux-gnupg].

### Backup Your Private Key

To backup your private key do the following:

```zsh
gpg --export-secret-keys --armor --output pgp-private-key.asc thealeos@gmail.com
```

You should store the backup in a really safe place. To import the backup when you need it:

```zsh
gpg --import pgp-private-key.asc
```

You might want to backup your revocation certificates located in `~/.gnupg/openpgp-revocs.d/`. 




[github-docs]: https://docs.github.com/en/authentication/managing-commit-signature-verification "Verify commit signatures"
[gnupg]: https://www.gnupg.org "GnuPG"
[invoking-gpg-agent]: https://www.gnupg.org/documentation/manuals/gnupg/Invoking-GPG_002dAGENT.html "Invoking GPG-AGENT"
[generating-a-new-gpg-key]: https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key "Generating a new GPG key"
[add-gpg-github]: https://docs.github.com/en/authentication/managing-commit-signature-verification/adding-a-gpg-key-to-your-github-account "Adding a GPG key to your GitHub account"
[archlinux-gnupg]: https://wiki.archlinux.org/title/GnuPG "GnuPG"

[^verify-commit-signatures]: Read more in the "[Verify commit signatures][github-docs]" section of GitHub Docs.
[^gpg-agent-env-requirements]: `gpg-agent` environment requirements: "[Invoking GPG-AGENT][invoking-gpg-agent]"
[^gen-new-gpg-key]: "[Generating a new GPG key][generating-a-new-gpg-key]" in GitHub Docs
[^add-gpg-github]: "[Adding a GPG key to your GitHub account][add-gpg-github]" in GitHub Docs
