
# fw-simple
Simple firewall on Debian 10 that configures **iptables** and **ipset** against SSH attempts, password failure immediately bans the user for 2 hours.

The solution takes ipset `sudo apt-get install ipset` and **inotifywait** `sudo apt-get install inotify-tools`.

- To load the script at reboot he `/etc/cron.d/perso` file contains (with an empty last line) :
```
# /etc/cron.d/perso

@reboot		root /root/fw-simple.sh > /dev/null 2>&1
```

- The script at `/root/fw-simple.sh` contains (with an empty last line) :

```sh
#!/bin/sh

IPSET_NAME="fw-simple"

export LANG=C
export PATH="/usr/bin:/usr/sbin"

ipset create "$IPSET_NAME" hash:net timeout 7200
iptables -P FORWARD DROP
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m set --match-set "$IPSET_NAME" src -j DROP

while inotifywait -e modify /var/log/auth.log; do
	str=$( tail -n1 '/var/log/auth.log' )
	if [ "$str" != "${str#*tion fail}" ]; then
		str="${str##* rhost=}"
		str="${str%% *}"
		check=str
		str="${str%.*}"
		if [ "$str" != "$check" ]; then
			str="${str%.*}"
			str="$str.0.0/16"
			ipset -exist add "$IPSET_NAME" "$str"
			echo "$str" >> /var/log/firewall
		fi
	fi
done
```

Use `sudo chmod 0700 /root/fw-simple.sh` and `sudo shutdown -r now` to initialize the process, and when the `/var/log/auth.log` file contains :
```
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=103.78.171.114 
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=95.26.110.114  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=167.71.197.212  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=165.22.101.34  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=106.12.123.199  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=210.100.165.51  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=119.28.113.42  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=165.154.33.41 
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=5.29.135.63  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=103.78.171.114  user=root
pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=103.187.146.93  user=root
```

Then `iptables -L -n --line-numbers` should show :
```
root@server-name:~# iptables -L -n --line-numbers
Chain FORWARD (policy DROP)
num  target     prot opt source               destination         

Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
2    DROP       all  --  0.0.0.0/0            0.0.0.0/0            match-set fw-simple src

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination       
```
And `ipset -L` should show :
```
root@server-name:~# ipset -L
Name: fw-simple
Type: hash:net
Revision: 6
Header: family inet hashsize 1024 maxelem 65536 timeout 7200
Size in memory: 1304
References: 1
Number of entries: 10
Members:
119.28.0.0/16 timeout 7128
103.78.0.0/16 timeout 7147
5.29.0.0/16 timeout 7141
165.154.0.0/16 timeout 7135
167.71.0.0/16 timeout 7097
106.12.0.0/16 timeout 7114
165.22.0.0/16 timeout 7104
95.26.0.0/16 timeout 7096
103.187.0.0/16 timeout 7171
210.100.0.0/16 timeout 7120

```

So the simple firewall is running.
