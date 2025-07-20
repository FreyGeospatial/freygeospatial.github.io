---
layout: post
title:  "Setup an Ubuntu Virtual Machine on MacOS with the UTM Emulator"
categories: [MacOS, Linux, shell scripting, virtual machines]
---

At work, I do most of my development in Ubuntu via WSL, and I’ve grown to really enjoy working in a Linux environment. For personal projects, I wanted to stick with Linux, but my home machine runs macOS. To bridge the gap, I turned to UTM — a free, open-source virtualization and emulation tool for macOS. While UTM is a great option and alternative to Parallels, setting up a Linux VM with it does involve a bit of extra effort.

In this post, we’ll walk through how to set up an Ubuntu VM in UTM and configure it so you can SSH into it from your macOS host. If you only need terminal access, SSH is recommended over using the UTM GUI for a smoother and more efficient experience.

1. install UTM
1. download your desired ubuntu instance (use ARM compatible ISO's for M-series Macs).
1. follow all the steps found here: https://www.youtube.com/watch?v=1PL-0-5BNXs
1. go to the UTM network settings for your new VM
1. change the "Network Mode" to "Emulated VLAN"
1. go to port forwarding and select these options: 

<div style="display: flex;">
    <div style="flex: 100%; text-align: center;">
        <img src="/images/utm/port-forwarding.png" alt="Port forwarding">
        <span style="font-size: 10px">Figure 1</span>
    </div>
</div>

1. start your ubuntu instance and log in
1. ensure you can allow for other machines to ssh into the Ubuntu guest machine by running 
    1. `sudo apt install openssh-server`
    1. `sudo systemctl enable ssh`
1. in your host terminal, run `ssh -p 22022 <username>@localhost` where `<username>` is your ubuntu username.
