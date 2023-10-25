# Build a working Chromium OS and a working devserver instance for an Over-The-Air update.md

## Task 0 Connectivity, tools & infrastructure readiness
### Deliverables
* My computer is an x86 architecture mackbook with Ubuntu22. 04 LTS installed via virtualbox.(At first I installed ubuntu 20.0 using parallels)
* I already have rented remote offshore cloud servers
* On the host I used clashX as proxy software and on the VM I used nat mode for access to google
### Gotchas & Challenges & Solutions
* At first, my computer itself had parallels and ubuntu 20.0 installed, after which I started to follow the Developed Guid until I encountered an error when executing repo init.
```
fatal: error unknown url type: https
fatal: cloning the git-repo repository failed, will remove '.repo/repo'
```
After that I checked the proxy, checked git, checked python, re-read the Develope Guide and tried to replace https with http, none of which worked, after which I retrieved the repo-discuss mailing list and pored over the init-related source code, and realized that this was a deeply hidden version-related issue, probably related to the version of git, python, Ubuntu, etc., While interested in what exactly is causing the problem, based on the purpose of the Develope Challenge，so I chose to give up in time and use VirtualBox to install Ubuntu22.04TLS, and the problem disappeared.
* Unable to open terminal
Just adjust the region and formatting to match and reboot and solved。
* Black screen when using the cros_sdk --create command.
```
[drm:vmw_host_log [vmwgfx]] *ERROR* Failed to send host log message
```
The virtualbox utility redistributes the disk size and solves the problem by entering the character interface and  repartitioning the hard disk.
```
VBoxManage modifyhd "/Users/fangzicheng/VirtualBox VMs/charles/charles.vdi" --resize 524288
fdisk /dev/sda
partprobe /dev/sda
```
## Task 1 Get Chromium OS code and start building 
### Deliverables
* Completed git, repo, google api, Gerrit authentication, chroot, etc. environment configuration
```
// config git
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

//add depot tool path
vim ~/.bashrc
export PATH=/home/charles/depot_tools:$PATH

//set umask
umask 022

//config Gerrit authentication
eval 'set +o history' 2>/dev/null || setopt HIST_IGNORE_SPACE 2>/dev/null
 touch ~/.gitcookies
 chmod 0600 ~/.gitcookies

 git config --global http.cookiefile ~/.gitcookies

 tr , \\t <<\__END__ >>~/.gitcookies
.googlesource.com,TRUE,/,TRUE,2147483647,o,git-liangzai97825.gmail.com=1//0g1bipgi-DlZNCgYIARAAGBASNwF-L9IrWWoOp9aEqoJ2v297BAVIY7MJnzd9tBpqyGRNlm9488SLEJJUVF8uRuEVceFjymV2jao
__END__
eval 'set -o history' 2>/dev/null || unsetopt HIST_IGNORE_SPACE 2>/dev/null


//set google api key
vim ~/.googleapikeys
google_api_keys = ""
google_default_client_id = ""
google_default_client_secret = ""

//config google storge
sudo apt-get install apt-transport-https ca-certificates gnupg curl sudo
echo "deb [signed-by=/usr/share/keyrings/cloud.google.asc] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo tee /usr/share/keyrings/cloud.google.asc
sudo apt-get update && sudo apt-get install google-cloud-cli
gcloud auth login
cp ~/.config/gcloud/legacy_credentials/*/.boto ~/.boto
```
* Already sync to release-R114-15437.B branch
```
  repo init -u https://chromium.googlesource.com/chromiumos/manifest -b release-R114-15437.B
  repo sync -j4
```
* Set board=amd64-generic and  build all packages
```
 cros_sdk --enter
 BOARD=amd64-generic
 
```
* Build packages and images
```
cros_sdk --enter
build_packages --board=${BOARD} --backtrack=1000
build_image --board=amd64-generic --no-enable-rootfs-verification test

```
* Finally Run in qemu & SSH & VNC
```
cros_vm --start --image-path=../../build/images/amd64-generic/R114-15437.75.0-d2023_10_25_165712-a1/chromiumos_test_image.bin --board=amd64-generic
```
![image](https://github.com/crescentBLADE/FOr-FydeOS-developer-challenge/blob/main/os.jpg)
![image](https://github.com/crescentBLADE/FOr-FydeOS-developer-challenge/blob/main/ssh.png)

### Gotchas & Challenges & Solutions
* Be careful ！Do not use sudo cros_sdk ! once you run it as root ,it might get you into a bad state that non-root cant recover! And you need to resync whole repo 
* No build_packages  options for cros
When I make the command **cros build-packages** in the official document, the terminal prompts that there is no such option, use -h to check the help, there is only the **build** option, use the build option prompts that the board is not set, and set the board prompts that the parameter is wrong. After adding the **debug** flag, I found that the essence of the call cros_sdk, and cros_sdk does not have a build option. After entering the trunck/src/scripts directory under the chroot environment, I did not find the build_packages script, and I did not find any related discussion in the mail list, and then I searched the root directory, and I found that there was a build_packages executable program in the bin directory, and the problem was solved.
```
charles@charles:~/chromiumos$ cros build-packages --board=${BOARD}
usage: cros [-h] [--log-level {fatal,critical,error,warning,notice,info,debug}] [--log-format LOG_FORMAT] [-v] [--debug]
            [--color] [--no-color] [--cache-dir CACHE_DIR]
            {analyze-image,ap,build,buildresult,chrome-sdk,cidbcreds,clean,clean-outdated-pkgs,debug,deploy,fix,flash,format,lint,query,shell,stage,try,tryjob,umount,unmount,workon,help}
            ...
cros: error: argument subcommand: invalid choice: 'build-packages' (choose from 'analyze-image', 'ap', 'build', 'buildresult', 'chrome-sdk', 'cidbcreds', 'clean', 'clean-outdated-pkgs', 'debug', 'deploy', 'fix', 'flash', 'format', 'lint', 'query', 'shell', 'stage', 'try', 'tryjob', 'umount', 'unmount', 'workon', 'help')

```
* build_packages failed
Based on the error message, i was found that this error originated from the Portage build tool.I want work out what "final slot" i want to have insalled，and is not work。
```
!!! Multiple package instances within a single package slot have been pulled
!!! into the dependency graph, resulting in a slot conflict
```
Then i use man command to get more  emerge command info，find this：
```
man emerge

       --backtrack=COUNT
              Specifies an integer number of times to backtrack if dependency calculation fails due to a conflict or an unsatisfied dependency (default: ´3´).
```
So  i use an insane number like 1000 to make sure nothing is missing or broken. And it works, finally build the packages.
## Task 2 Kernel replacement
### Deliverables
* In the project documentation it asks to replace the kernel with version 5.15, then when I checked the src/overlays/overlay-amd64-generic/profiles/base/make.defaults file I found that the version defaults to 5.15, and for the betterment of the challenge I chose to upgrade the kernel version to 5.4
* After carefully reviewing the Portage package management system and cros_sdk, I successfully replaced the original kernel and installed it remotely
```
cros_workon --board=${BOARD} start sys-kernel/chromeos-kernel-5_4
# remove old kernel 
emerge-amd64-generic --unmerge chromeos-kernel-5_15

# buildkernel 
cd ~/trunk/src/third_party/kernel/v5.4
FEATURES="noclean" cros_workon_make --board=${BOARD} --install chromeos-kernel-5_4

# Configure Kernel
./chromeos/scripts/kernelconfig editconfig

# add kernel
emerge-${BOARD} sys-kernel/chromeos-kernel-5_4


# remote install kernel
~/trunk/src/scripts/update_kernel.sh --remote 127.0.0.1 --ssh_port 9222 
```
![image](https://github.com/crescentBLADE/FOr-FydeOS-developer-challenge/blob/main/update.jpg)
And it works great .  
## Task 3 CrOS devserver in docker
### Deliverables
> view all the acronyms  https://chromium.googlesource.com/chromiumos/docs/+/HEAD/glossary.md#acronyms
* run devserver locally
### Gotchas & Challenges & Solutions
* run start_devserver error ,got this error
```
 File "/usr/bin/start_devserver", line 1446, in <module>
    main()
  File "/usr/bin/start_devserver", line 1398, in main
    common_util.SymlinkFile("/build", pkgroot_dir)
  File "/usr/lib64/python3.6/site-packages/chromite/lib/xbuddy/common_util.py", line 297, in SymlinkFile
    os.symlink(target, link_name)
PermissionError: [Errno 13] Permission denied: '/build' -> '/usr/bin/static/pkgroot7u25xsb0-link'
```
Started with Sudo the dev server run
* As per the requirement, the devserver needs to be deployed in docker, I have read a lot and did not understand how to deploy the devserver separately, but made some attempts
 According to the official docs, the prerequisite for using devserver is to have done build_packages, so I built a custom docker image, installed the relevant dependencies, depot tool, and mounted the chromium directory into the image(because of the docker design , unable to mount local files use dockerfile,just can use -v flag for run command), and did repo init and repo sync in the image, which went well, but cros_sdk --enter and cros_create both failed.
```
FROM ubuntu:22.04
//dockerfile
RUN apt-get update && apt-get install -y software-properties-common
RUN add-apt-repository universe && apt-get install -y git  curl xz-utils
RUN useradd -ms /bin/bash myuser
RUN usermod -aG sudo myuser
USER myuser
WORKDIR /home/myuser
RUN git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git && echo "PATH=/home/myuser/depot_tools:$PATH" >> ~/.bashrc
RUN mkdir -p ~/chromiumos
WORKDIR ~/chromiumos

//run command
  docker build -t mydevserver:v3 . 
  docker run -it -v /home/charles/chromiumos:/home/myuser/chromiumos mydevserver:v3 /bin/bash (or docker run -d && docker exec -it )
```
There's no time to continue more attempts now, so I'll have to skip it for now, but I suspect it has something to do with missing cros_sdk related dependencies, and circular dependencies on the chroot environment 
## Task 4 Connecting the dots (use local devserver not docker)
### Deliverables
* change lsb 
```
cros_set_lsb_release --board=amd64-generic --auvserver http://127.0.0.1:8282/update --devserver http://127.0.0.1:8282 --version_string=15437.75.0 --sysroot=/etc
```
* run devserver success
* set static_dir and update success
```
sudo start_devserver --port 8282 --static_dir ~/
```
### Gotchas & Challenges & Solutions
* cant edit /etc/lsb-release ,according to the official docs,can be overridden in /mnt/stateful_partition/etc/lsb-release,but there is no such file。
* I learned from the official documentation that the script gradually shifted to the bin directory under chromite, found cros_set_lsb_release, and after getting help with it via -h, successfully set up the lsb-release

## summarize
So far still many options to try with improvements and figuring out why, but for many reasons such as machine performance issues, strangeness of chromiumos ecosystem, and time, my challenge can only go so far,but i had a good time . Thanks for reading!
