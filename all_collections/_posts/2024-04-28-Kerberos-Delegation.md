---
layout: post
title: Kerberos Delegation  
date: 2024-04-28
categories: ["Kerberos", "Delegation", "Attack", "Active Directory","Cybersecurity",]
---
<style>*{direction: rtl}</style>
<!-- Text can be **bold**, _italic_, ~~strikethrough~~ or `keyword` -->

<!-- [Link to another page](./another-page.html). -->



المقالة تشرح واحد من اهم المفاهيم في بيئات الـ Active Directory وهو مفهوم Kerberos Delegation حيث يعتبر البوابة لنقل المستخدمين واعطائهم حق الوصول لـ اصول الشبكة دون الحاجة للوصول بشكل مباشر من قبلهم, يقدم Kerberos Delegation ثلاث انواع للتفويض تم تناول تفاصيل عملها وامكانية استغلال كل نوع .

> المقالة تتطلب فهم كامل لطريقة عمل الـ  Kerberos  + NTLM  authentication protocols

التفويض يحل مشكلة معروفة بالـ Double Hop وب اختصار هي محاولة الوصول باستخدام معلومات الاعتماد الخاصة بك (credentials) بصورة NTLM او Ticket, والاستفادة من موارد الاصول في الشبكة بناء على نوع التفويض المتاح. 

![del](../../img/posts/post_kerb-Del/del.png)

الصورة السابقة توضح عمل التفويض افتراضا ان هنالك خادم File Share يتم التعامل معه عبر خادم ويب لا يمكن الوصول المباشر لخادم File Share من قبل المستخدمين عند تسجيل الدخول كـ يوزر test على جهاز المستخدم (USER PC) -   (Interactive authentication) يتم تخزين الـ credentials الخاصة بـ test في الـ lsass.exe لاستخدامها فيما بعد, عند محاولة الدخول على موقع الويب المستضاف على خادم الويب يقوم USER PC بعمل
 Network Authentication باستخدام credentials الخاصة بـ test المخزنة مسبقا داخل الـ lsass.exe

> لا يتم تخزين الـ credentials في الخادم المراد الوصول له في حالة  Network Authentication.

![dell](../../img/posts/post_kerb-Del/dell.png)

الصورة السابقة توضح انه في حال عدم تفعيل الـ Delegation على

WEB Server سيتم رفض الاتصال من قبل الـ File Share والسبب يعود ان الـ WEB Server لا يملك اي بيانات اعتماد للمستخدم لارسالها لخادم المشاركة.

قامت مايكروسوفت بداية من Windows 2000 باطلاق اول نوع من انواع التفويض Unconstrained Delegation يليه Constrained Delegation في ـ Windows Server 2003 ثم في Windows 2012 تم تقديم 
Resource Based Constrained Delegation (RBCD) الـ RBCD متفق في العمل مختلف في مكان اعداده, حيث انه يتم على الخادم الاخير في حالتنا يتم اعداده على ال File Share ومنه يتم تحديد الاصول التي تملك صلاحية التفويض من عدمه . 


> تتطلب الانواع Unconstrained و Constrained نوع خاص من الصلاحيات SeEnableDelegationPrivilege يتم اعطائه لـ اليوزر القائم على تشغيل خدمة الويب,  يمنح في العادة لـ enterprise and domain admins.

![delll](../../img/posts/post_kerb-Del/delll.png)

![con](../../img/posts/post_kerb-Del/con-all-uncon.png)

![after](../../img/posts/post_kerb-Del/after-auth.png)

كما هو واضح في الصورة السابقة عند تفعيل التفويض يستطيع WEB Server مخاطبة الـ DC لطلب Ticket من نوع (Service Ticket) ST نيابة عن المستخدم test واستخدامها للوصول الى خادم المشاركة بـ اعتباره المستخدم test. 

>للإختصار سنتجاوز شرح عمل البروتوكلات الخاصة بالتحقق (SSP) 



---

<h1 style="text-align:center; direction:ltr;">
UnConstrained Delegation
</h1> 

يعد النوع الاول من التفويض الاسهل في الاعداد والاسهل في الاستغلال ايضا حيث لإعداده يتطلب الامر تفعيل
 “trust this computer for delegation to any service“ 


![after](../../img/posts/post_kerb-Del/uncon/uncon-sett.png)

بعد التفعيل يجب ان يحصل الـ Machine Account على الصلاحية الموضحة بالاسفل  

![after](../../img/posts/post_kerb-Del/uncon/uncon-powerviw.png)

> الحساب WEB-1$ هو الحساب القائم على تشغيل خدمات الويب.

عند طلب المستخدم test الوصول الى ملف home الموجود على خادم المشاركة عن طريق خادم الويب يكون الطلب بالترتيب الموجود في الصورة اسفل..

![after](../../img/posts/post_kerb-Del/uncon/wire-uncon.png)
![after](../../img/posts/post_kerb-Del/wire-ucnon1.png)

>- DC      IP : 10.10.10.10
>- WEB-1 IP : 10.10.10.12
>- FS-1    IP : 10.10.10.15
>- Client IP : 10.10.10.100

1.  1- طلب TGT من  قبل المستخدم 
1.  2- طلب ST للوصول الى خادم الويب 
1.  3- طلب  TGT اضافية لارسالها الى خادم الويب لاستخدامها بالنيابة عن المستخدم
1. 4- يتم تمرير الـ ST الى خادم الويب التي تحمل بداخلها الـ TGT الاضافية, لتصفح home/ يعرف الطلب هذا بـ AP-REQ
1. 5- بما ان home/ متواجد على خادم اخر في هذه الحالة fs-1 يقوم خادم الويب بطلب ST للمستخدم test من الـ DC للوصول الى fs-1 باستخدام ال TGT المرفقة له في الخطوة رقم 4.
1. 6-يقوم خادم الويب بتمرير ST الى خادم المشاركة نيابة عن المستخدم 
1. 7-يقوم خادم الويب ب إرجاع بيانات المجلد للمستخدم test  يعرف بـ AP-REP

>ماذا لو اردنا طلب ST  لـ اي خادم اخر في الشبكة ..؟

تكمن خطورة هذا النوع من التفويض في طريقة عمله حيث انه يقوم بتخزين تذكرة المستخدم المراد التفويض له مثلا المستخدم test عند استخدام خادم الويب سيقوم خادم الويب بتخزين TGT Ticket الخاصة بالمستخدم test لاستخدامها لمخاطبة الـ DC لطلب (Service Ticket) ST نيابه عن المستخدم  لتمريرها لخادم المشاركة .. النوع هذا من التفويض غير مقيد اي انه عند الاحتفاظ بنسخة من الـ TGT Ticket, تستطيع طلب ST لـ اي خدمة على اي خادم.

عند اختراق خادم الويب نستطيع استغلال التفويض وتكون درجة الخطورة بناء على
TGT Tickets المحفوظة عليه تعود الى اي مستخدم, مثلا يتناول سيناريو الاختراق بالاسفل استغلال TGT Ticket محفوظة,
لـ Domain Admin account - (admin-dom) ..

>جميع سيناريو الاختراقات المذكورة في المقال يتم تطبيقها عن طريق Sliver C2 + Rubeus وبعض الادوات المساعدة.


![after](../../img/posts/post_kerb-Del/uncon/sliver-uncon1.png)
يمكننا الامر triage باستعراض TGT المحفوظة على الخادم في الصورة تم ايجاد تذكرة تعود للمستخدم admin-dom بعد البحث عن المستخدم كما هو موضح بالاسفل تبين ان المستخدم ينتمي لمجموعة الـ Domain Admins.
![after](../../img/posts/post_kerb-Del/uncon/domain.png)
![after](../../img/posts/post_kerb-Del/uncon/sliver-uncon2.png)


تم استخراج التذكرة الخاصة بالمستخدم بصيغة Base64

![after](../../img/posts/post_kerb-Del/uncon/uncon-to-chache.png)

الصورة بالاعلى يتم فيها تحويل التذكرة من Base64 وحفظها بصيغة ccache

![after](../../img/posts/post_kerb-Del/uncon/uncon-to-kirbi.png)

ثم استخدام impacket-ticketConverter  لتحويل التذكرة من صيغة ccache الى صيغة Kirbi 



![after](../../img/posts/post_kerb-Del/uncon/uncon-export-kirbi.png)
![after](../../img/posts/post_kerb-Del/uncon/uncon-klist.png)

بعد استخدام export لاستخدام التذكرة في نظام لينكس يمكنك التحقق منها عبر الامر klist .



بعد الانتهاء من اعداد استخراج التذكرة الخاصة بالمستخدم admin-dom وتهيئتها للاستخدام وبعد المعرفة بانه مستخدم ذو صلاحية عالية يمكننا الاتصال باستخدام التذكرة مباشرة بالـ DC باستخدام, psexec كما هو موضح بالاسفل . 



![after](../../img/posts/post_kerb-Del/uncon/uncon-psexec-dc01.png)
>يتم تمرير k -no-pass- لعدم طلب كلمة مرور واستخدام التذكرة الموجودة مسبقا .



---


<h1 style="text-align:center; direction:ltr;">
Constrained Delegation
</h1>

التفويض المقيد يوفر مميزات مهمة اولها تقليل الخطورة فهو لا يقوم بتخزين الـ TGT الخاصة بالمستخدمين ايضا يوفر الدعم للمصادقة باستخدام NTLM عكس Unconstrained فهو لا يقبل الا Kerberos يعتمد الـ Constrained في عمله على Service-for-User (S4U) Kerberos extensions , تستطيع اعداده  Kerberos only او Protocol transition 



>Kerberos only : the service can delegate when the client authenticates using Kerberos
(uses S4U2Proxy) 

>Protocol transition : the service can delegate regardless of how the client authenticates (uses S4U2Self and S4U2Proxy)

<h2 style="text-align:center; direction:ltr;">
Kerberos only
</h2>

نبدا بالـ Kerberos only حيث يتم اعداده لتحقيق نفس الهدف وهو الوصول غير المباشر والاستفادة من موارد الخوادم عن طريق خادم وسيط يملك حق التفويض, لاعداد التفويض المقيد كالتالي 

![after](../../img/posts/post_kerb-Del/con1/con-ker-sett.png)
![after](../../img/posts/post_kerb-Del/con1/con-powerview.png)

يجب تحديد ماهي الخدمة المراد الوصول لها وعلى اي خادم وسيتم رفض محاولة الوصول لـ اي خدمة اخرى غير محددة مسبقا, في مثالنا نريد الوصول لخدمات المشاركة يجب تحديد الخدمة CIFS على الخادم fs-1.0xs.lab , عند طلب المستخدم test الوصول الى ملف home الموجود على خادم المشاركة عن طريق خادم الويب يكون الطلب بالترتيب الموجود في الصورة اسفل..

![after](../../img/posts/post_kerb-Del/con1/wire-con.png)
![after](../../img/posts/post_kerb-Del/wire-ucnon1.png)

>- DC      IP : 10.10.10.10
>- WEB-1 IP : 10.10.10.12
>- FS-1    IP : 10.10.10.15
>- Client IP : 10.10.10.100

1. 1- طلب TGT من قبل المستخدم 
1. 2- طلب ST للوصول الى خادم الويب 
1. 3- يتم تمرير الـ ST الى خادم الويب, لتصفح home/ يعرف الطلب هذا بـ
AP-REQ
1. 4- بما ان home/ متواجد على خادم اخر في هذه الحالة fs-1 يقوم خادم الويب بعمل S4U2Proxy وهي عبارة عن ارسال الـ ST الخاصة باليوزر التي تم طلبها في الخطوة رقم 2 + TGT الخاصة بـ (WEB-1$)Machine Account لـ DC
لطلب ST لخادم المشاركة بالنيابة عن المستخدم.
1. 5- يقوم خادم الويب بتمرير ST الى خادم المشاركة نيابة عن المستخدم 
1. 6- يقوم خادم الويب بارجاع بيانات المجلد للمستخدم test  يعرف بـ AP-REP


<h2 style="text-align:center; direction:ltr;">
Protocol transition
</h2>
لإعداد هذا النوع يجب وضع الإعدادات التالية ايضا يتطلب تحديد نوع الخدمة والخادم

![after](../../img/posts/post_kerb-Del/con1/con-any-sett.png)

![after](../../img/posts/post_kerb-Del/con1/con-any-powerview.png)

في النوع السابق يتم استخدام ST الخاصة بالمستخدم + TGT الخاصة بالـ

Machine Account للإرسالها لـ DC لطلب ST للمستخدم للدخول على مجلد المشاركة هذا مفهوم S4U2Proxy, ماذا اذا كانت طريقة الدخول على خادم الويب عن طريق NTLM Protocol. 





لن يكون هنالك ST لاستخدامها لارسال S4U2Proxy نحتاج لـ اضافة اخرى تعطينى القدرة على طلب ST لخادم الويب نستطيع استخدامها لتنفيذ S4U2Proxy لطلب ST لخادم المشاركة, الـ S4U2Self تعطينى القدرة على ذلك, الترتيب في الاسفل يمثل تسجيل الدخول على خادم الويب عن طريق  NTLM.

![after](../../img/posts/post_kerb-Del/con1/wire-con-any.png)
![after](../../img/posts/post_kerb-Del/wire-ucnon1.png)

>- DC      IP : 10.10.10.10
>- WEB-1 IP : 10.10.10.12
>- FS-1    IP : 10.10.10.15
>- Client IP : 10.10.10.100

1. 1- تتم المصادقة عبر NTLM الى خادم الويب
1. 2- يتم طلب ST لـ خادم الويب عبر تنفيذ S4U2Self 
1. 3- يتم استخدام الـ ST من الخطوة رقم 2 لاستخدامها لتنفيذ S4U2Proxy لطلب ST لـ خادم المشاركة بالنيابة عن المستخدم 
1. 4- يقوم خادم الويب بتمرير ST من الخطوة رقم 3 الى خادم المشاركة نيابة عن المستخدم 
1. 5- يقوم خادم الويب بارجاع بيانات المجلد للمستخدم test



يتم استغلال التفويض المقيد في الغالب لتصعيد الصلاحيات والانتقال من خادم الى خادم اخر بصلاحيات اي مستخدم نريد, شرط ان تكون الخوادم محددة مسبقا في اعدادات التفويض.

![after](../../img/posts/post_kerb-Del/con1/sliver-triage-ker.png)

![after](../../img/posts/post_kerb-Del/con1/sliver-dump-tgt-machine.png)

بالصورة السابقة بعد اختراق خادم الويب تم استخراج تذكرة TGT الخاصة بـ WEB-1$ لاستخدامها في تنفيذ S4U2Proxy 

![after](../../img/posts/post_kerb-Del/con1/sliver-pass-the-tgs.png)



 >impersonateuser: The user we want to impersonate.
 >msdsspn: The service principal name that WEB-1 is allowed to delegate to.
 >user: The principal allowed to perform the delegation.
 >ticket: The TGT for /user.
 >ptt: To inject the ticket in session.
 




باستخدام Rubeus s4u يتم تنفيذ S4U2Self ثم S4U2Proxy نيابة عن مستخدم نختاره في حالتنا admin-dom 


![after](../../img/posts/post_kerb-Del/con1/sliver-after-s4u.png)

![after](../../img/posts/post_kerb-Del/con1/klist.png)

بعد تنفيذ الهجوم نرى وجود ST للمستخدم admin-dom لـ خادم المشاركة fs-1 تمكنك من الانتقال له بصلاحيات عالية .



---

<h1 style="text-align:center; direction:ltr;">
Resource Based Constrained Delegation (RBCD)
</h1>

يتم تطبيق النوع الاخير من انواع التفويض على الخادم المراد الوصول له في حالتنا (fs-1) File Share يقوم بتحديد الخوادم المسموح لها بالتفويض له في هذه الحاله يحدد الـ (web-1)WEB server.





لا يتم تحديد نوع الخدمة كسابقة تستطيع طلب اي خدمة تريدها عليه ايضا لا يتم تحديد البروتوكول فهو يقبل NTLM و Kerberos يعتمد بشكل مباشر على نوع الإتصال ويتصرف بناء عليه ان كان NTLM سيقوم بتنفيذ 
 S4U2Self and S4U2Proxy وفي حالة Kerberos  سيتم تنفيذ S4U2Proxy .. يمكن إعداد الخادم المراد تفعيل النوع هذا عليه كالتالي :


>Setting up this delegation does not require Domain or Enterprise Admin privileges
• Just write rights over the msDS-AllowedToActOnBehalfOfOtherIdentity attribute of a service account in our case (fs-1)

![after](../../img/posts/post_kerb-Del/RBCD/checktheSID.png)

يجب ان نحصل على SID الخاص بالخادم الذي سيعطى صلاحية التفويض  

![after](../../img/posts/post_kerb-Del/RBCD/EditTheBinary.png)

بعد الحصول على web-1 SID نستطيع الكتابة على 
msDS-AllowedToActOnBehalfOfOtherIdentity في خادم المشاركة عن طريق الامر في الصورة السابقة.

![after](../../img/posts/post_kerb-Del/RBCD/CheckTheStatusRBCD.png)

ترتيب الاتصال في الشبكة في حالة NTLM سيتبع الترتيب السابق وايضا في حالة Kerberos, استغلال هذا النوع ايضا كالسابق,  يصنف الهجوم كـ واحد من هجمات تصعيد الصلاحيات والتنقل في الشبكة.


>طريقة الهجوم القادمة غير مرتبطة في ماقبلها, تحاكي الهجمة تفعيل تفويض RBCD لتصعيد الصلاحيات 


عند التمكن من الكتابة على اي Computer Object يجب ان ياتي RBCD كـ احد الطرق لتصعيد الصلاحية المثال القادم يفترض اننا استطعنا الكتابة على 
(Computer Object (win10 بعد اختراقه ولكن بصلاحيات مستخدم طبيعي, يمكننا انشاء Machine Account وتفعيل الـ  RBCD على win10 واعطاء صلاحية التفويض لـ Machine Account المنشئة من قبلنا .

![after](../../img/posts/post_kerb-Del/RBCD/cuota.png)

يعتمد هذا الهجوم كليا على القدرة على انشاء Machine Account الاعدادات الافتراضية في الـ Active Directory يمكن لكل مستخدم حق انشاء حتى 10 Machine Account.

![after](../../img/posts/post_kerb-Del/RBCD/make.png)

![after](../../img/posts/post_kerb-Del/RBCD/evil.png)

تم استخدام الاداة StandIn.exe لإنشاء الـ Machine Account 

![after](../../img/posts/post_kerb-Del/RBCD/cal-pass.png)

العملية السابقة هي لتحويل كلمة مرور الحساب الى hash لاستخدامها في طلب الـ TGT الخاصة فيه.

![after](../../img/posts/post_kerb-Del/RBCD/evilsid.png)

![after](../../img/posts/post_kerb-Del/RBCD/RBCD-win10.png)

![after](../../img/posts/post_kerb-Del/RBCD/checkwin10.png)

بعد ان تم انشاء الـ Machine Account الخطوات السابقة توضح عملية تفعيل الـ RBCD على ال win10 ليعطي صلاحية التفويض لحساب EvilComputer 

![after](../../img/posts/post_kerb-Del/RBCD/tgt-evil.png)

طلب TGT خاصة بالحساب EvilComputer 

![after](../../img/posts/post_kerb-Del/RBCD/s4u.png)

الصورة السابقة توضح طلب التفويض للمستخدم admin-dom لـ win10 حيث تم تنفيذ S4U2Self + S4U2Proxy من قبل EvilComputer وطلب ST لـ admin-dom لاستخدامها على win10.

![after](../../img/posts/post_kerb-Del/RBCD/klist.png)


يعد التفويض من الخدمات المهمة وتعتبر الاصول التي يتم تطبيق التفويض عليها بجميع انواعه من الاصول الحساسة جدا .. حيث يمكن للمهاجم عند اختراقها استغلال التفويض لتحقيق اهدافه مثل ما تم استعراضه سابقا . 

إنتهى …. 

