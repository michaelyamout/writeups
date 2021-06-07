## Hack The Box - Jeeves (10.10.10.63)

To begin with, we will start an nmap scan on our target as part of our first enumeration steps. For this, I will opt to use nmapAutomator, although you can run a traditional nmap scan. 

### Enumeration

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
'PORT      STATE SERVICE      VERSION
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
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows'


**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```
We have four open ports on our target machine. After no success with enumerating port 445 (SMB) I decide to open our target machine in a web browser to view what is hosted on port 80 (http://10.10.10.63:80). It appears our target is running ‘Ask Jeeves’ as a service and is running on ‘Microsoft-IIS/10.0’. I will ran a DirBuster scan to enumerate furthur files and directories here but did not obtain anything useful. 


### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/michaelyamout/writeups/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
