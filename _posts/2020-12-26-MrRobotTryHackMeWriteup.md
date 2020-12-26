---
layout: post
title: "Mr. Robot CTF – Try Hack Me Writeup"
---

If you haven’t already, I highly suggest attempting this Try Hack Me room. Personally, I’ve never watched the show (I know shame on me!), but I’ve heard good things about it. The room is on the easier side but it is great for beginner-intermediate attackers to sharpen their skills and learn something new. First let’s start with the normal nmap scan. I like using the -sC -sV options to grab a bunch of information. -sC is a script scan that grabs information like http headers and more. -sV grabs the versions of each service. The -oN writes the output to a file which I named “initial”. Another scan I like to run includes the -A and -T4 options. The -A is a very aggressive scan that is loud and runs -O (OS detection), -sV, -sC, and --traceroute. You can always do a ‘man nmap’ and look through the different options. Remember, only run these scans on hosts you have permission to scan.

<h2>Scanning</h2>
 
![nmapscan](/images/MrRobotPics/Capture.PNG)
We see that ports 80 and 443 are both open (HTTP and HTTPS). Going to the site is cool but it doesn’t really reveal much information. Next, I set off a dirbuster scan. You can also run a gobuster scan for the same effect (gobuster dir -u http://<boxIP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt). If you follow this file path on kali linux there are more wordlists including the ‘rockyou.txt’ wordlist.
 
![Dirbuster](/images/MrRobotPics/Capture1.PNG)

The important directories that we find are the:
-	/wp-login
-	/readme
-	/robots.txt

<h2>Gaining Access</h2>
The readme directory had a little bit of information basically telling us that we are on the right track. If we traverse to the robots directory we find two files that we can grab:
 
![robots](/images/MrRobotPics/Capture2.PNG)

The key file is of course flag #1 of the challenge.
 
![key1](/images/MrRobotPics/Capture3.PNG)

![savefsocity](/images/MrRobotPics/Capture4.PNG)

If we traverse to the fsocity.dic file and save it, we see that it is a wordlist.
 
![fsocity](/images/MrRobotPics/Capture5.PNG)

Next, I looked at the word press login page.
 
![wp-login](/images/MrRobotPics/Capture6.PNG)

Now we have a wordlist and a login page. Any ideas what we do next? That’s right we are going to brute force credentials.
 
![burp](/images/MrRobotPics/Capture7.PNG)

I used burp suite to grab the request for the login page and then used hydra to attempt to brute force the username and password. In the above pictures I attempt a login with username ‘admin’ with password ‘password’. As we can see from the webpage screenshot the server responds with ‘Invalid username’ which we can use in our hydra brute force.
 
![hydra](/images/MrRobotPics/Capture8.PNG)

We found the username Elliot. I believe that this is also one of the names of the characters in the show. So if you know the show you probably could’ve guessed the username. When we try to login with the username Elliot we get the error ‘The password you entered for the username Elliot is incorrect’ This we can use in our password brute force. 
 
![failedlogin](/images/MrRobotPics/Capture11.PNG)

![hydra2](/images/MrRobotPics/Capture9.PNG)

After waiting for a long time for the hydra attack to work I tried editing the wordlist by eliminating duplicates and then tried again. 
 
![editingfsocity](/images/MrRobotPics/Capture12.PNG)

![edit2](/images/MrRobotPics/Capture13.PNG)

This still didn’t work (took too long) so I gave up on hydra. I’m sure if I allocated more RAM to my virtual machine or had better hardware this should work. I started researching other ways to brute force a wordpress login page and I stumbled upon the tool wpscan.  WPScan is a tool that grabs a ton of information from word press pages and even has password brute force capabilities.
 
![wpscan](/images/MrRobotPics/Capture14.PNG)

![password](/images/MrRobotPics/Capture15.PNG)

Boom I found the password. This found the password much faster for me. The password is 9 characters long, but I have blotted it out here. Now we can login to the page with this username and password!
 
![homepage](/images/MrRobotPics/Capture30.PNG)

Searching around the page for a little bit for file upload I found the editor page. This page lists all the pages on the sight and their source code. Soon an idea formed. If I can request a page, couldn’t I trigger the page code to deliver me a shell. Php-reverse-shell by pentestmonkey works perfectly fine here. This php file is located in kali linux in directory /usr/share/webshells or on the web at pentestmonkey at ([http://pentestmonkey.net/tools/web-shells/php-reverse-shell](http://pentestmonkey.net/tools/web-shells/php-reverse-shell)) for download. 

![phpreverseshell](/images/MrRobotPics/Capture16.PNG)

When we open it we see that we have to edit the local host IP address and what port you want to connect back to. I set my Try Hack Me IP which can be found on tun0 with the ‘ip a’ command. Now we just need to copy this php code and paste it into one of the php pages with the editor. I fully replaced the php code when I did this. I also chose the 404.php page as my target but you can pick whatever page you want to go to and don’t forget to save it.
 
![editthemes](/images/MrRobotPics/Capture17.PNG)

Now we spin up our netcat listener on the port we set the reverse shell to.
 
![netcat](/images/MrRobotPics/Capture18.PNG)

Then we traverse to a page on the server that does not exist (or whatever page you put your reverse shell in).
 
![error404](/images/MrRobotPics/Capture19.PNG)

And boom we now have a shell on the box.
 
![shell](/images/MrRobotPics/Capture20.PNG)

![filesystem](/images/MrRobotPics/Capture21.PNG)

Snooping around the file system a little bit I found the key.txt file in the home folder of user ‘robot’. But we see that we cannot access it.
 
![bashshell](/images/MrRobotPics/Capture22.PNG)

Above is another picture of the netcat connection but this time I transferred it to a bash shell using the oneliner $ python -c “import pty; pty.spawn(‘/bin/bash’);” . We also see a file in the folder called password.raw-md5. If we cat that file we see a user and a hash of the password. 
 
![passwordcrack](/images/MrRobotPics/Capture23edited.PNG)

Using the password cracker called CrackStation we can crack this md5 hash to get the password. Then we just ‘su robot’ and enter the password to get to that user. Now we can cat the key.txt!
 
![key2](/images/MrRobotPics/Capture24edited.PNG)

Boom. Well, what now? Now we try to get root privileges. I perused around the file system a little bit with ‘ls -la’ to see what permissions ‘robot’ had but I did not find much. I researched some common privilege escalation techniques and found the following one liner. 
-	find / -perm /4000 -print 2>/dev/null
 
![permissions](/images/MrRobotPics/Capture26.PNG)

I found that there is nmap on this machine running in /usr/local/bin which is runnable by user ‘robot’. Nmap is an amazing tool. I decided to grab what version it was running. And found that the version on the server was very old compared to the version I’m running on my kali instance. 
 
![nmaptotherescue](/images/MrRobotPics/Capture27.PNG)

I then did some googling “nmap version 3.81 exploit” and found the link ([https://pentestlab.blog/category/privilege-escalation/](https://pentestlab.blog/category/privilege-escalation/)). This included more permission commands and also something called nmap --interactive:
-	find / -user root -perm -4000 -print 2>/dev/null
-	find / -perm -u=s -type f 2>/dev/null
-	find / -user root -perm -4000 -exec ls -ldb {} \;
-	nmap --interactive
The find command as you can probably tell is a very powerful command in Linux for finding files with different permissions.
 
![nmap](/images/MrRobotPics/Capture28.PNG)

So I tried the nmap --interactive command and made it bash by using the ‘!sh’ command. Boom now I am root and I can grab the final flag.
 
![nmap2](/images/MrRobotPics/Capture29edited.PNG)

Congrats you did it!

