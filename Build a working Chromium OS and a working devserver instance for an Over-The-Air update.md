# Build a working Chromium OS and a working devserver instance for an Over-The-Air update.md

## Task 0 Connectivity, tools & infrastructure readiness
### Deliverables
* My computer is an x86 architecture mackbook with Ubuntu22. 04 LTS installed via virtualbox.(At first I installed ubuntu 20.0 using parallels, see the specific description in task 1 for the reason for the replacement)
* I already have rented remote offshore cloud servers
* On the host I used clashX as proxy software and on the VM I used nat mode for access to google

## Task 1 Get Chromium OS code and start building 
### Deliverables
* Completed git, repo, google api, Gerrit authentication, chroot, etc. environment configuration
* Already sync to release-R114-15437.B branch
```
  repo init -u https://chromium.googlesource.com/chromiumos/manifest -b release-R114-15437.B
  repo sync -j4
```
### Gotchas & Challenges & Solutions
* At first, my computer itself had parallels and ubuntu 20.0 installed, after which I started to follow the Developed Guid until I encountered an error when executing repo init.
```
fatal: error unknown url type: https
fatal: cloning the git-repo repository failed, will remove '.repo/repo'
```
After that I checked the proxy, checked git, checked python, re-read the Develope Guide and tried to replace https with http, none of which worked, after which I retrieved the repo-discuss mailing list and pored over the init-related source code, and realized that this was a deeply hidden version-related issue, probably related to the version of git, python, Ubuntu, etc., so I chose to give up in time and use VirtualBox to install Ubuntu22.04TLS, and the problem disappeared.

