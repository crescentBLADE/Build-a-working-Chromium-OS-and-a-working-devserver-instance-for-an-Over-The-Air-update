# Build a working Chromium OS and a working devserver instance for an Over-The-Air update.md

## Task 0 Connectivity, tools & infrastructure readiness
### Deliverables
* My computer is an x86 architecture mackbook with Ubuntu22. 04 LTS installed via virtualbox.(At first I installed ubuntu 20.0 using parallels, see the specific description in task 1 for the reason for the replacement)
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
build_pasckages --board=${BOARD}
build_image --board=amd64-generic --no-enable-rootfs-verification test

```
* Finally Run in qemu & SSH & VNC
### Gotchas & Challenges & Solutions

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
