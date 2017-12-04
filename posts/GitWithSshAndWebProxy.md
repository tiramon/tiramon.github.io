# Access git repository through ssh behind a webproxy

## Preconditions
- You got a git repository somewhere that is reachable via git
- You are in a network that uses a webproxy

## The Problem
When you are in a network that uses a http proxy and no ssh port is open you normally can't reach your git repository in the internet

## The Solution
Actually there are multiple solutions to this problem

1. Tunnel with putty
	- Download [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
	- To connect to a ssh server outside the network configure a valid ssh connection in PuTTY and additionally goto Connection -> Proxy in the PuTTY Configuration. Set 'Proxy type' to 'HTTP' and add the proxy informations in 'Proxy hostname' and 'Port'
	- To tunnel your git requests through that connection goto Connection -> SSH -> Tunnels and add 'Source port' 22 and the name of the host plus port of the git repository to Destination i.e. 'www.tiramon.de:22'
	- Now you can clone your repository with ``git clone localhost:/path/to/repository``

2. Go directly through the proxy
I'm using Windows 10 and just had to add following lines to `~/.ssh/config`

```bash
Host www.tiramon.de
  ProxyCommand connect -H my.proxy.url:8080 %h %p
```

or if you are using gitbash you could also add it to `/etc/ssh/ssh_config`