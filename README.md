
# Description
It runs sshuttle from WSL2, and sets up the routing table of Windows.
Since it modifies the routing table of Windows,
the admin privilege of Windows is required.
(Open WSL2 terminal with "Run as Administrator".)

Routing behavior:
- Excluded hosts (-x) are routed through their currently-resolved best
  route (longest prefix match), so pre-existing VPN routes (e.g. OpenVPN)
  pointing at them are preserved. When no route covers an excluded host,
  the Windows default gateway is used as fallback.
- Pre-existing routes on the tunneled subnets are left intact: wsshuttle
  only adds its own routes (metric 1, so they win), and on cleanup
  deletes only the routes it added itself (by specifying the gateway).

# Requirements
- iproute2 (`ip`), util-linux (`column`)
- bash >= 4.0
- sshuttle >= 1.1.1 (tested with 1.3.x)

# Installation
```bash
sudo apt install -y ipcalc iptables
pip3 install sshuttle==1.1.1 # Other version may cause "doas" error.
curl https://raw.githubusercontent.com/vchirikov/wsshuttle/main/wsshuttle | sudo install /dev/stdin /usr/local/bin/wsshuttle
```

# Usage
```bash
wsshuttle --help
wsshuttle [--delete] [--dry] [--upgrade] [--version] <sshuttle_args...>
```

# Examples

Caution!: Admin privilege of Windows is required.

Route all (0/0) packets through ssh-server except destination is 157.0.0.0/8.  
Specify the IP address of ssh-server in -x as well, otherwise it will be
tunneled too.

```bash
wsshuttle -r ssh-server -x 3.3.3.3 -x 157.0.0.0/8 0/0
```

The same with the long option (short `-x`/`-r`/`-l`/`-e`/`-i` and long
`--exclude`/`--include`/`--listen`/`--remote`/`--ssh-cmd` are both recognized
for routing):

```bash
wsshuttle -r ssh-server -x 3.3.3.3 -x 157.0.0.0/8 0/0
```

Deletes the routes added by wsshuttle.  
If you have any problem, do it.

```bash
wsshuttle -r ssh-server -x 3.3.3.3 -x 157.0.0.0/8 0/0 --delete
```

Dry-run, just prints commands.  
Doesn't make any changes.

```bash
wsshuttle -r ssh-server -x 3.3.3.3 -x 157.0.0.0/8 0/0 --dry
route.exe delete 3.3.3.3     mask 255.255.255.255 192.168.3.1      if 7
route.exe delete 157.0.0.0   mask 255.0.0.0       192.168.3.1      if 7
route.exe delete 0.0.0.0     mask 128.0.0.0       172.18.187.223   if 46
route.exe delete 128.0.0.0   mask 128.0.0.0       172.18.187.223   if 46
route.exe add    3.3.3.3     mask 255.255.255.255 192.168.3.1      metric 1 if 7
route.exe add    157.0.0.0   mask 255.0.0.0       192.168.3.1      metric 1 if 7
route.exe add    0.0.0.0     mask 128.0.0.0       172.18.187.223   metric 1 if 46
route.exe add    128.0.0.0   mask 128.0.0.0       172.18.187.223   metric 1 if 46
sshuttle -l 0.0.0.0:0 -x 3.3.3.3 -x 157.0.0.0/8 -r ssh-server -x 3.3.3.3 -x 157.0.0.0/8 0/0
route.exe delete 3.3.3.3     mask 255.255.255.255 192.168.3.1      if 7
route.exe delete 157.0.0.0   mask 255.0.0.0       192.168.3.1      if 7
route.exe delete 0.0.0.0     mask 128.0.0.0       172.18.187.223   if 46
route.exe delete 128.0.0.0   mask 128.0.0.0       172.18.187.223   if 46
```

The excludes are routed through their currently-resolved best route.
For example, if OpenVPN has a route `203.0.113.0/24 -> 203.0.113.1 if 9`,
then `-x 203.0.113.10` becomes
`route.exe add 203.0.113.10 mask 255.255.255.255 203.0.113.1 metric 1 if 9`
instead of being forced through the Windows default gateway (as in the original wsshuttle)
e.g. the OpenVPN routing keeps working. The delete commands are scoped by the gateway as well, 
so only the routes added by wsshuttle are removed.

While running, wsshuttle also temporarily lowers the interface metric of the WSL vEthernet (e.g. 5000 -> 5). Windows adds the interface metric to the route metric, so without this the tunneled subnet routes 
(metric 1 + 5000) would lose to competing routes like OpenVPN's (metric 1 + 25) and traffic would bypass the tunnel. The previous value is restored on exit.

# Recommended Settings

Invoke 'wsshuttle' when hit 'sshuttle'.  
When not admin, do privilege elevation automatically. ~~
Installing 'sudo' on PowerShell is required (scoop install sudo)

```bash
sshuttle(){
    if net.exe session &>/dev/null; then # if admin
        $SHELL -ic "wsshuttle $*"
    else
        powershell.exe sudo wsl $SHELL -ic \"wsshuttle $*\"
    fi
}
```

