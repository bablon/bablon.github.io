---
layout: post
title:  "Secure Boot, Kernel and Drivers"
date:   2023-11-04 20:30:00
categories: kernel
---

## Secure Boot

现在Linux发行版安装和使用时已不需要在BIOS关闭安全启动，因为Bootloader和内核已被认证方签过名。但是如果安装自己编译的内核和驱动，仍然会遇到secure boot问题，因为它们没有签名。Linux提供了机器级别的机制来签名，通过该机制，就不用关闭安全启动了，只需要在开发或者安装时添加签名步骤。

以下操作以Debian 12为基础。

## Machine Owner Key

首先创建Machine Owner Key, 它是签名的基础。创建的Key保存在/var/lib/shim-signed/mok目录下MOK.priv, 证书是MOK.der或者MOK.pem。

    # mkdir -p /var/lib/shim-signed/mok/
    # cd /var/lib/shim-signed/mok/
    # openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -days 36500 -subj "/CN=My Name/"
    # openssl x509 -inform der -in MOK.der -out MOK.pem

第二要注册创建的Key, 记住提示输入的一次性密码。

    $ sudo mokutil --import /var/lib/shim-signed/mok/MOK.der

最后重启机器，在启动时按照提示完成注册。

    $ sudo mokutil --test-key /var/lib/shim-signed/mok/MOK.der
    /var/lib/shim-signed/mok/MOK.der is already enrolled

## Kernel

sbsign和sbverify用于签名和检查。创建脚本sign-kernel简化内核签名，用法`sign-kernel <version>`。

    ver=$1
    shim=/var/lib/shim-signed/mok/

    if [ -f /boot/vmlinuz-$ver ]; then
        sudo sbsign --key $shim/MOK.priv --cert $shim/MOK.pem /boot/vmlinuz-$ver --output /tmp/vmlinuz-$ver.tmp
        if [ $? -ne 0 ]; then
            echo "failed to sign kernel $ver"
            exit 1
        fi
        sudo mv /tmp/vmlinuz-$ver.tmp /boot/vmlinuz-$ver

        sbverify --cert $shim/MOK.pem /boot/vmlinuz-$ver
        if [ $? -ne 0 ]; then exit 1; fi
    else
        echo "failed to find kernel $ver"
        exit 1
    fi

## Drivers

创建脚本sign-module给驱动签名，用法`sign-module <module> [module ...]`。

    ksrc=/lib/modules/$(uname -r)/build/
    sign=$ksrc/scripts/sign-file
    shim=/var/lib/shim-signed
    key=$shim/mok/MOK.priv
    pub=$shim/mok/MOK.pem

    if [ -z "$1" ]; then exit 1; fi

    echo -n "Enter PEM pass phrase: "
    read -s KBUILD_SIGN_PIN
    export KBUILD_SIGN_PIN
    echo

    for file in $*; do
        sudo --preserve-env=KBUILD_SIGN_PIN $sign sha256 $key $pub $file
        if [ $? -ne 0 ]; then
            echo "failed to sign $file"
            exit 1
        fi
    done

## DKMS

dkms安装驱动时创建和使用了不同的MOK，/var/lib/dkms/目录下mok.key和mod.pub，同样要注册后才有效。

若要使用上面创建并已注册的MOK，可在配置文件中修改添加配置。

    mok_signing_key="/var/lib/shim-signed/mok/MOK.priv"
    mok_certificate="/var/lib/shim-signed/mok/MOK.der"

如果MOK有密码，dkms安装时失败，需要通过`KBUILD_SIGN_PIN`变量传入密码。如安装`v4l2loopback-dkms`：

    read -s KBUILD_SIGN_PIN
    export KBUILD_SIGN_PIN
    sudo --preserve-env=KBUILD_SIGN_PIN apt install v4l2loopback-dkms

## Reference

Debian Secure Boot [Wiki](https://wiki.debian.org/SecureBoot)
