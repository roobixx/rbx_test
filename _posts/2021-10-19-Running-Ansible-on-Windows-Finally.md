---
title: 'Finally Running Ansible from Windows...kinda.' 
layout: posts
author_profile: true
---


## Disclaimer: I consider everything below to be a hack. It works but it is a hack for sure. Enjoy 

---

## Can you run Anisble on Windows?

Not according to Ansible...

![Ansible FAQ](/assets/images/win_ansible/ansible_windows.png)

## See...

![Ansible on Windows?](/assets/images/win_ansible/ansible-playbook_not_in_PATH.png)

For many years I have been frustrated with the inability to run Ansible playbooks natively in Windows. Yes, Ansible works well in Windows Subsystem for Linux (WSL), but that comes with some limitations. Well one major limitation for myself. I could not use ansible as a provisioner in Packer from Windows. This is especially problematic if you want to build Hyper-V based images from WSL. To the best of my knowledge, you cannot do this because the user Packer is running as, needs to be in the ```'Hyper-V Administrators' or 'Administrators'```groups.

![Hyper-V Admin](/assets/images/win_ansible/hyper-v_group.png)

So what do you do???

Honestly, I have given up on running Ansible on Windows many times over, but then something would pop up and I would try to make it work and fail and give up again. I never liked the idea of running packer, then spinning the VM back up to run ansible - there had to be a better way. Well, a couple of weeks ago I needed to try to make Packer, Ansible, and Hyper-V play nice on a Windows host, so I dove back in.

## Batch Scripts FTW?!?

In my research to see if someone else had managed to figure out a decent solution, I stumbled upon a GitHub gist that looked very promising. It did not take me long to realize what they were doing and how it all worked. That gist can be found [here](https://gist.github.com/zaenk/20178e637299404dcdcc1ff4667f24d7).


What user [zaenk](https://github.com/zaenk) had done was create a collection of batch scripts that when put in the ```PATH``` on a Windows machine, would be called from Packer during the provisioning process, translate the slashes appropriately from Windows to Linux and passing everything as a command argument to Bash.exe. This was the missing link for me. A way to call ansible from WSL while in the Windows context.


I copied the ```.cmd``` files to my local machine and added them to my PATH and fired off a packer build. Sadly, it was not long before things started to error, and I was not getting the results I had hoped for.

![playbook output](/assets/images/win_ansible/playbook_garbage.png)

After a bit of troubleshooting, mainly have the batch script ```ansible-playbook.cmd``` output what it was passing to ```C:\Windows\System32\bash.exe -c``` as a command argument to a text file. Upon doing this it became clear that there was a lot of issues with escaping characters and others not being properly escaped. Eventually I came up with a version of ```ansible-playbook.cmd``` that was passing the now escaped arguments from Packer to ```ansible-playbook``` within WSL.

![Output](/assets/images/win_ansible/output_garbage.png)


This is what I came up with for and 'alias' for ansible-playbook

```
@echo off
REM "alias" for packer ansible provisioner running in cmd/powershell
set args=%*
REM replace backslashes with slashes
call set args=%%args:\=/%%
REM fix drive letters
call set args=%%args:C:=/mnt/c%%
REM call set args=%%args:D:=/mnt/d%%
REM call set args=%%args:E:=/mnt/e%%
call set args=%%args:/"=%%
call set args=%%args:"'='%%
call set args=%%args:'"='%%
call set args=%%args:"=^'%%

C:\Windows\System32\bash.exe -c "ansible-playbook %args%"
```

![Build Successful](/assets/images/win_ansible/build.png)

Flow:

  - Packer is invoked to build VM template
  - ```ansible-playbook.cmd``` is ran when Packer calls ```ansible-playbook```
  - ```ansible-playbook.cmd``` rewrites all the slashes and escapes all the quotes necessary for WSL
  - ```ansible-playbook.cmd``` Runs bash.exe and passes escaped arguments from Packer to ```ansible-playbook``` in WSL where normal execution starts.

### Issues

There are some issues with how the output is displayed from ansible-playbook and I will look into resolving those but for the time being, this solution works and meets my needs. Just wish I had it 3-4 years ago.

## More details

Can be found on [Github](https://github.com/roobixx/AnsibleOnWindows) including a script for running ansible stand-alone.
