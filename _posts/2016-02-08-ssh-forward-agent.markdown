---
layout: post
title: "SSH: ForwardAgent v ProxyCommand"
date: 2016-02-08
categories:
---
### What is ForwardAgent actually doing, and should I be using it?

An interesting discussion went around one of our tech mailing lists the other day regarding ForwardAgent in .ssh/config. This was initially sparked by an issue where authentication failures were being triggered due to too many public keys being offered. 

This semi derailed into a discussion around AgentForwarding and ProxyCommand directives in .ssh/config. After reading through some of the mailing list responses and having a nose around online I found a couple of handy resources that explain the difference between these directives and why you might want to utilise ProxyCommand instead of AgentForwarding when connecting to hosts via a jump/bastion server. 

Let's assume we are using a jump server to connect from our workstation to server C. We cannot connect directly to server C due to network restrictions. 

We have two options here. Use AgentForwarding or use ProxyCommand. 

Utilising AgentForwarding is the 'easy' way to achieve the desired access. In this case the ssh agent will connect to the jump  server and forwards the agent on to this server, this effectively creates a file socket used by the ssh-agent to communicate with other processes. This will then be set as an environment variable of $SSH_AUTH_SOCK that contains the path to this socket (/tmp/xxxx/agent.xxx). Steve Frield has a really cool breakdown of how ssh authentication methods work located [here](http://www.unixwiz.net/techtips/ssh-agent-forwarding.html) which was really handy in understanding what exactly AgentForwarding is doing. 

While we will never have the private key on the jump  server, it is the ssh-agent that provides authentication responses from our workstation. That is to say it has access to the private key on our workstation. This leaves it vulnerable to interception/hijacking. Since any jump server is presumably a multi-user environment there could be other users with root access that could manipulate files on the server. A simple explanation of a real example of this is provided by David Schoen on his [blog](http://blog.lyte.id.au/) which I've excerpted below.

>"In practice one way attackers might use this is if they can see the environment variables your SSH connection came in via (either they have gained access to your account or root) then they could interrogate the SSH\_AUTH_SOCK environment variable, add that in to their own environment and use your authentication agent to initiate authenticated connections to other hosts while you are still connected to the untrustworthy host."

It's prudent to trust our shared jump environment less than we trust our own workstation security (multiple users may have root privileges over this shared server and we may not have oversight of who has this access). Given this, we would ideally like to avoid a potential attack vector by utilising the ProxyCommand directive instead. We can add this to our .ssh/config file for all connections and it will look something like: 


```
 Host *
        ProxyCommand ssh -aY <jumpserverip/hostname> 'nc -w 900 %h %p'
        ForwardAgent no
```

From this you can see that we are using netcat to create a tunnel through which our client will connect so by utilising ProxyCommand we end up with end to end encryption without exposing our ssh-agent on our jump server. 

I liked Keith Herron's explanation with diagrams that you can read on his [blog](http://backdrift.org/transparent-proxy-with-ssh)

Interestingly it seems like there's plenty of places that suggest using AgentForwarding, for example github has a pretty neat tutorial  [tutorial](https://developer.github.com/guides/using-ssh-agent-forwarding/) for setting this up.  However there's also plenty of discussion as to why this isn't a good idea and ProxyCommand should be used instead, Johannes Gilger has a good discussion at his [blog](https://heipei.github.io/2015/02/26/SSH-Agent-Forwarding-considered-harmful/) that's good reading as well. 

I definitely didn't have much of an opinion on this prior to reading through the mailing list responses but it seems to me from looking into it a little bit that generally speaking one should be preferencing using ProxyCommand unless there's a compelling reason to need to utilise AgentForwarding. 

###Reading List
* https://developer.github.com/guides/using-ssh-agent-forwarding/
* http://www.unixwiz.net/techtips/ssh-agent-forwarding.html
* https://heipei.github.io/2015/02/26/SSH-Agent-Forwarding-considered-harmful/
* http://blog.lyte.id.au/2012/03/19/ssh-agent-forwarding-is-a-bug/
* http://backdrift.org/transparent-proxy-with-ssh

