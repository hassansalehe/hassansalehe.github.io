---
layout: post
title: "Turning your windows 10 machine into Ubuntu environment"
date: 30-04-2017
comments: true
---
#### Introduction
In the past, there were two options you could access Ubuntu in 
your machine which runs Windows OS as as a default operating system:
1. Creating a dual boot with Ubuntu
2. Using Virtualization

Nowadays, if you have Windows 10, you can benefit from the Beta 
Ubuntu feature that ships with Win10. I will show the steps on 
how to get it to working full.

#### The Steps
1. [Jack explains how to turn on the Ubuntu beta feature on 
Windows 10.](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide). 
Once you turn the Ubuntu feature, go to step 2.
2. [Install ConEmu, a better, multi-feature terminal emulator](https://conemu.github.io/)
3. Install developement tools
  * I have compiled a small list of tools you can install on your Ubuntu account to 
    make a full development environment.
    ```bash
    sudo apt-get install git
    sudo apt-get install -y subversion
    sudo apt-get install -y ruby-full
    sudo apt-get install -y libdwarf-dev
    sudo apt -y install duff dos2unix ccrypt apcalc sfftw-dev # sfftw: discrete fourier transform
    sudo apt-get install -y lib32z1 lib32ncurses5
    sudo apt-get install -y flex bison build-essential csh libxaw7-dev
    sudo apt-get install -y qemu # to run ARM code: qemu-arm <your_binary>
    sudo apt-get install -y unace rar unrar zip unzip p7zip-full p7zip-rar sharutils 
    ```
  4. [Add bash/terminal to select from right-click context.](http://www.windowscentral.com/how-launch-bash-shell-right-click-context-menu-windows-10), [2](https://blog.cyplo.net/posts/2016/07/06/terminal-emulator-windows-10-bash.html)
