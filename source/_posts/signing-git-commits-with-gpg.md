---
title: 使用 GPG 签名认证 git 提交
date: 2020-07-26 09:43:46
tags: git
---

## Install GPG

### MacOS

``` bash
brew install pinentry-mac gpg2
touch  ~/.gnupg/gpg-agent.conf
echo "pinentry-program /usr/local/bin/pinentry-mac" >> ~/.gnupg/gpg-agent.conf
```

### Archlinux

``` bash
yay -S gunpg
```

## Configure

### Generate key

``` bash
gpg --full-gen-key
```

### List keys

``` bash
gpg --list-secret-keys --keyid-format LONG

# /home/username/.gnupg/pubring.kbx
# -------------------------------------
# sec   rsa4096/1111111111111111 2019-08-07 [SC]
#       2222222222222222222222221111111111111111
# uid                 [ultimate] realname (comment) <example@gmail.com>
# ssb   rsa4096/3333333333333333 2019-08-07 [E]
```

### Export public key

``` bash
gpg --armor --export 1111111111111111

# -----BEGIN PGP PUBLIC KEY BLOCK-----

# CONTENT
# -----END PGP PUBLIC KEY BLOCK-----
```

### Register public key

将上面的 `public key` 拷贝到 [github settings](https://github.com/settings/keys)

## Associating your GPG key with Git

### Set gpg key

``` bash
git config --local user.signingKey 1111111111111111
```

### Signing commits

在提交时，使用 `-S` 标志

``` bash
git commit -S -m 'commit message'
```

或者通过配置避免每次都要输入 `-S`

``` bash
git config --global commit.gpgsign true
```

## FQA

1. gpg: signing failed: Inappropriate ioctl for device

    ``` bash
    echo "export GPG_TTY=$(tty)" >> ~/.bashrc
    ```

1. GPG Hangs When Private Keys are Accessed

    ``` bash
    gpgconf --kill gpg-agent
    ```

1. `secret key not available` or `gpg: signing failed: secret key not available`

    ``` bash
    git config --global gpg.program gpg2
    ```
