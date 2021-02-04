---
layout: post
title: "SystemD, The D is for Dumb"
date: 2018-05-15
categories: [linux]
---

A few Fridays ago I bumped into what can only be called an extremely annoying 'feature' of systemd that started breaking the process by which we manage users.
I'll preface this by pointing out that I'm aware that ansible plays are a horrible way to deal with user management and some kind of pull based config management like puppet is clearly pererable for this kind of task That being said, I haven't had the chance to look into the puppet module code for user management, so perhaps this also affects it. (I'm going to take a guess here and say it doesn't).

## Setting the scene
If I want to remove a user, my current process is to run an ansible play against the hosts that I want the user yanked from. Part of the reason we rely on ansible plays to deal with this is as there's some adiditional application level interaction that needs to happen since we don't just create local system accounts for users but also a number of application specific logins and configurations that vary according to the categorisation fo the host roles.
With that in mind, for the OS side and local logins we utilise the user module that's build into ansible. By now a large portion of the fleet is updated to utilise a kernel version that includes systemd.

## It's not a bug, it's a feature
Turns out that in recent systemd implementations when your user logs out systemd doesn't actually stop all the processes related to that user. Ansible's user module checks for running processes under that user and if it finds processes it fails (this is because it's essentially just running a userdel on the local machine, and userdel doesn't handle running processes started under that user, it just errors out). The first issue was that our playbook doesn't echo full logs, because no_log is set to prevent us doing things like echoing out user passwords. Fine, so set log to enable, which can be done with no_log:
```
root@pluto:~# ps aux | grep bob
bob 22712 0.0 0.0 45272 4744 ? Ss Apr05 0:00 /lib/systemd/systemd --user
bob 22714 0.0 0.0 63652 2332 ? S Apr05 0:00 (sd-pam)
```
Okay weird. The user was logged out so why are those processes still in use? Turns out this is a 'feature' of systemd. A quick google found a couple of bug reports across different distributions that all utilise the same underlying systemd architecture and then we can see some discussion around the actual issue here: (https://github.com/systemd/systemd/issues/2975)

The summary of this discussion is that systemd utilises /etc/logind.conf which has this configuration variable:
```
KillUserProcesses=No
```
As the default. This basically means, if the user leaves some processes running don't kill them on logout. Normally you'd expect on logout most user related processes to die, however looking through those bug reports and pull requests it seems like they couldn't seem to get systemd to be able to viably distinguish between user login and environment processes and processes like tmux or screen. It ended up that you can either set this value to true and then when your user logs out screen and tmux sessions die (not ideal) or you keep it to false and then you end up with these kind of ghost processes.

It's not necessarily a big deal but there were a bunch of similarly related reports where people were confused as to why new configus didn't come into affect, and part of this was that systemd wa sholding user related processes open and thus not restarting them on next login and picking up changes. The 'fix' for this is to reboot, but for those of us runnning large infrastructure where we don't frequently reboot (a debate for another day) this isn't feasible, especially on AWS instances that utilise ephemeral storage.

## Fixing the feature
So the obvious answer is to kill the processes which works on a single case by case basis. However, when managing a fleet of upwards of a couple of thousand instances I can't be manually ssh'ing and killing processes every time i need to delete a user. We could also have the ansible play narrow down pids cusing problems and kill these, but i don't really like the idea of doing a ps and grepping out all pids being run by that user. It feels like that's open to error. There's another way though...

We could add this to our plays.. Prior to to calling the user module to remove the user we can invoke a local chell command and run 'systemctl stop user@'$UID'.server. A nice easy way to get this is to use id -u so something like this
```
systemctl stop user$`id -u $USER`.service
```
That's going to ensure that systemd has stopped those user processes prior to any attempt to delete the user. Of course there's downsides to this such as 'what if that user was running something that was really important' in which case i guess you have a larger question around why local users that have SSH access are running vital processes instead of these being owned by service users.

Anyway it seems like this isn't going to be 'fixed' any time soon and we didn't want to just implement killing user processes on logout since we do actually utilise screen and tmux to some degree on the estate.
