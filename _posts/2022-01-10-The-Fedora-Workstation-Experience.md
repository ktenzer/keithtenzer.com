--- 
layout: single
title:  "The Fedora Workstation Experience"
categories:
- Linux
tags:
- Linux
- Fedora
- Opensource
---
![Fedora](/assets/2022-01-10/desktop.png)
## Overview
In 2022 choice matters, talk is even cheaper and action is many decibels louder than it used to be. In addition, more and more people are realizing the power of voting with their wallets (ESG) to make sure ideals or values are represented by what is purchased. The more we collectively choose something the more of it will naturally exist and be successful. Many of us of course, share opensource ideas and principals, believing an open world is a better world. Now I am not advocating our world should be void of closed systems and proprietary software just given the choice and things being similar, I choose opensource. I am also not advocating, I nor anyone else judge other's for their decisions. We never truly understand the circumstances of someone elses decision because we aren't in those shoes. I am merely trying to bring awareness to the decision itself, in this case wether to install Linux or Windows/MacOS and of course show how you how I configure my workstation with Fedora 34.

Being an ex-macbooker myself, I love the simplicity and minimalist approach of Apple's UI design and experience. As such I aim to find the best of both worlds: great UI experience without compromising on freedom and choice!

## Hardware
Currently I am running Fedora 34 on the Lenovo Thinkpad X1 but great thing here is you have choice. I get that Apple's Macbook's look and feel great (touchpad works great) but those are pretty minor things and your going to cover your laptop in stickers anyway, right? 

## Installing Fedora Latest
Fedora has a [media writer](https://getfedora.org/en/workstation/download/) and you can download this to create a USB Image to install Fedora. You can provide your own ISO or just let it create an image using the latest Fedora. Once you have USB image simply reboot your laptop/desktop press F12 or F8 to enter boot menu and boot from your USB. The rest is just accepting defaults. If your more advanced user you may want to also setup your own filesystem partitions and I always recommend turning on disk encryption with LUKS.

## Customizing Fedora Workstation
Once you get booted into Fedora it's time to customize. Unlike MacOS or Windows you can truly customize the desktop environment and while that is powerful and rewarding it also can be time consuming as well as turn most people off. The point here is to get something great with limited effort.

### Update Fedora
Just like any OS clicking the update button is usually the first step. You can of course click on Applications->System Tools->Software which launches the UI software package manager but this is Linux right?

<pre>$ sudo dnf update -y</pre>

![Fedora](/assets/2022-01-10/update.png)

### Install GNOME Extentions
GNOME is the UI operating environment and has a modular plugin framework for extending it. There have been huge advances in GNOME performance and while developers do all the hard work, users make those contributions meaningful which in turns leads to more development resources. 
<pre>$ sudo dnf install gnome-extensions-app</pre>

![GNOME Extentions](/assets/2022-01-10/extentions.png)

#### Install Dock from Dash Extensions
This extension will add a docking bar in center for favorite applications just like MacOS.
Navigate to [https://extensions.gnome.org/](https://extensions.gnome.org/) and in search filled enter "dock from dash". Click on the extension and in upper right there is a slider to enable the extension.

![Dock and Dash](/assets/2022-01-10/dock_and_dash.png)

#### Install GNOME Tweaks
Using GNOME tweaks you can configure many aspects of the UI we will be using later

<pre>$ sudo dnf install -y gnome-tweaks</pre>

### Install Google Chrome

<pre>$ sudo dnf install google-chrome</pre>

### Disable Wayland
Overall wayland works but one area there is still issues is in screen sharing. If you care about that I would recommend disabling it.

<pre>
$ sudo vi /etc/gdm/custom.conf
WaylandEnable=false
</pre>

### Configure MacOS Theme
These steps will enable MacOS icons and desktop theme.

<pre>$ sudo dnf install la-capitaine-icon-theme</pre>

<pre>$ git clone https://github.com/paullinuxthemer/Mc-OS-themes.git</pre>

<pre>$ mkdir ~/.themes</pre>

<pre>$ cp -r Mc-OS-themes/McOS-MJV ~/.themes</pre>

Open GNOME Tweak tool Applications->Utilities->Tweak and navigate to appearance section. Set application theme to McOS-MJV and icons to La-Capitane. Navigate to window titlebars section and enable maximize/minimize under titlebar buttons. This adds buttons to all windows that let you maximize or minimize them.

![Themes](/assets/2022-01-10/themes.png)

![Appearance](/assets/2022-01-10/appearance.png)

![Window Buttons](/assets/2022-01-10/minimize_buttons.png)

### Install RPM Fusion
This is a tool that gives you access to a lot of community developed tools and is very useful for workstations.

<pre>$ sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm</pre>

### Optimize Battery Usage
There are additional drivers needed to ensure your battery is used efficiently. Not installing these leads to quicker battery drain.

<pre>$ sudo dnf install tlp tlp-rdw</pre>

For thinkpads add additional driver from RPM fusion.
<pre>$ dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm
</pre>

### Install Multimedia Codecs
<pre>$ sudo dnf groupupdate multimedia --setop="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin</pre>
<pre>$ sudo dnf groupupdate sound-and-video</pre>

### Setup Solarized
Solarized is a color scheme for terminal sessions. Let's face it staring at a black screen with green text is bad and tiring for your eyes. These color patters are soothing and will let you enjoy starring at CLI terminals.
<pre>$ sudo dnf install -y p7zip</pre>
<pre>$ mkdir ~/solarized</pre>
<pre>$ cd ~/solarized</pre>
<pre>$ git clone https://github.com/sigurdga/gnome-terminal-colors-solarized.git</pre>
<pre>$ gnome-terminal-colors-solarized/install.sh</pre>

Setup vim and install solarized colors for vim.
<pre>$ sudo dnf install -y vim</pre>
<pre>$ git clone https://github.com/altercation/vim-colors-solarized.git</pre>
<pre>$ mkdir -p ~/.vim/colors</pre>
<pre>$ cp vim-colors-solarized/colors/solarized.vim ~/.vim/colors/</pre>

### Install VSCODE
Pretty much the standard IDE for software development, infrastructure-as-code, blogging or anything that requires editing text.
<pre>$ sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc</pre>

<pre>$ cat <<EOF | sudo tee /etc/yum.repos.d/vscode.repo
[code]
name=Visual Studio Code
baseurl=https://packages.microsoft.com/yumrepos/vscode
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc
EOF
</pre>

<pre>$ sudo dnf check-update</pre>
<pre>$ sudo dnf install -y code</pre>

### Install Ansible (Optional)
If you aren't automating stuff with Ansible it is never to late to change your life.
<pre>$ sudo dnf install -y ansible</pre>
<pre>$ sudo dnf install -y python-ansible-runner</pre>
<pre>$ pip install ansible-runner-http</pre>
<pre>$ pip install openshift</pre>
<pre>$ ansible-galaxy collection install kubernetes.core</pre>

### Install Go (Optional)
Golang is by far my language of choice for it's simplicity and elegance.
<pre>$ mkdir -p ~/go/src/github.com</pre>
<pre>$ sudo dnf install -y go</pre>
<pre>$ vi ~/.bash_profile
export GOBIN=/home/username
export GOPATH=/home/username/src
export EDITOR=vim
</pre>

### Install OpenShift and Kubectl (Optional)
Likely there are newer releases so grab latest.
[OpenShift and Kubectl 4.9](https://access.redhat.com/downloads/content/290/ver=4.9/rhel---8/4.9.13/x86_64/product-software)

### Install Operator SDK (Optional)
If you want to put all those Ansible skills to use in cloud-native world start writing Operators.

Likely there are newer releases so grab latest or release for your OCP release.
[Operator Framework 4.9](https://docs.openshift.com/container-platform/4.9/operators/operator_sdk/osdk-installing-cli.html#osdk-installing-cli-linux-macos_osdk-installing-cli)

## Summary
In this article we discussed the importance of choice and at least why you might consider Linux for your next workstation operating system. We also go through a step-by-step guide in configuring your Fedora workstation and give it similar look and feel to MacOS. If I only convinced one person to give Linux a try then it was all worth it!

(c) 2022 Keith Tenzer




