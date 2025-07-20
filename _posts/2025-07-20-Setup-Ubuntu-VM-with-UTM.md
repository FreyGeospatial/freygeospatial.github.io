---
layout: post
title:  "Setup an Ubuntu Virtual Machine on MacOS with the UTM Emulator"
categories: [MacOS, Linux, shell scripting, virtual machines]
---

At work, I do all my development work in Ubuntu through WSL- the Windows Subsystem for Linux. I figured it would be useful to have some virtual machines handy for running Linux on my own personal machine as a bit of a development sandbox. Plus, it would allow me to run software that I might not be able to with containers; a VM can emulate hardware after all, whereas a container cannot.

I run MacOS, and so I decided to download UTM, which is open source and free (though charges a nominal fee to support development through the Apple App Store). But while free, there is some slight overhead in setting up a VM with it.

So, we will quickly go over setting up an Ubuntu instance on UTM and allowing us to ssh into it from the MacOS host (using ssh is recommended over the UTM GUI if only requiring use of the terminal).

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
