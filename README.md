from pathlib import Path

readme = r"""# سناریوی تمرینی: ساخت Internal Developer Platform (IDP)

> **هدف:** ساخت یک پلتفرم توسعه‌دهنده داخلی روی Linux و Kubernetes واقعی؛ به‌طوری‌که یک توسعه‌دهنده بتواند بدون درخواست دستی از تیم DevOps، سرویس جدیدی بسازد، آن را deploy کند و وضعیتش را ببیند.

## فهرست مطالب

- [۱. سناریوی کاربر](#۱-سناریوی-کاربر)
- [۲. چرا این سناریو را اجرا می‌کنیم؟](#۲-چرا-این-سناریو-را-اجرا-میکنیم)
- [۳. معرفی ابزارها](#۳-معرفی-ابزارها)
- [۴. زیرساخت پایه](#۴-زیرساخت-پایه)
- [۵. فازهای اجرا](#۵-فازهای-اجرا)
- [۶. تست نهایی](#۶-تست-نهایی-end-to-end)
- [۷. ابزارهای مکمل](#۷-ابزارهای-مکمل)
- [۸. ترتیب اولیه اجرا](#۸-ترتیب-اولیه-اجرا)
- [۹. زیرساخت اصلی: Linux و Kubernetes واقعی](#۹-زیرساخت-اصلی-linux-و-kubernetes-واقعی)
- [۱۰. فهرست کامل ابزارها](#۱۰-فهرست-کامل-ابزارها)
- [۱۱. چالش‌های هر ابزار](#۱۱-چالشهای-هر-ابزار)
- [۱۲. سناریوی End-to-End و Game Day](#۱۲-سناریوی-end-to-end-و-game-day)
- [۱۳. ترتیب نهایی اجرا](#۱۳-ترتیب-نهایی-اجرا)
- [۱۴. مشاهده‌پذیری و OpenTelemetry](#۱۴-مشاهدهپذیری-و-opentelemetry)
- [۱۵. ابزارهای تکمیلی: Vault، Loki، PostgreSQL و Cost](#۱۵-ابزارهای-تکمیلی-vault-loki-postgresql-و-cost)

---

## ۱. سناریوی کاربر

### وضعیت فعلی

سارا، توسعه‌دهنده جدید شرکت فرضی «کمپانی»، می‌خواهد سرویسی به نام `payment-service` بسازد. در وضعیت فعلی باید از تیم DevOps درخواست کند تا:

- یک repository بسازد؛
- CI/CD pipeline را تنظیم کند؛
- manifestهای Kubernetes را آماده کند؛
- دسترسی RabbitMQ را فراهم کند؛
- برای ابزارهای مختلف حساب کاربری جداگانه ایجاد کند.

این فرایند ممکن است چند روز طول بکشد.

### وضعیت هدف

سارا فقط یک‌بار با حساب خودش در Keycloak وارد سیستم می‌شود و از طریق SSO به Backstage و ابزارهای مجاز دسترسی پیدا می‌کند.

1. در Backstage گزینه **Create** را انتخاب می‌کند.
2. قالب **Python Service** را انتخاب می‌کند.
3. نام `payment-service` را وارد و تأیید می‌کند.
4. Backstage یک repository جدید در Git می‌سازد که شامل کد پایه، manifestها و فایل `catalog-info.yaml` است.
5. سرویس در Backstage Catalog ثبت می‌شود.
6. Argo CD تغییر Git را تشخیص می‌دهد و سرویس را deploy می‌کند.
7. سرویس به RabbitMQ وصل می‌شود و پیام ارسال و دریافت می‌کند.
8. سارا وضعیت سرویس و وضعیت Sync آن را از صفحه سرویس در Backstage می‌بیند.

### پشت صحنه

- Vault و External Secrets Operator، secretهای موردنیاز را به Kubernetes می‌رسانند؛ هیچ secretای در Git ذخیره نمی‌شود.
- Keycloak و Backstage از PostgreSQL مدیریت‌شده توسط CloudNativePG استفاده می‌کنند.
- OpenTelemetry traceها را تولید می‌کند؛ Tempo آن‌ها را ذخیره می‌کند و Loki لاگ‌ها را نگه می‌دارد.
- Grafana محل مشاهده traceها، لاگ‌ها و metricها است.
- OpenCost هزینه تقریبی منابع مصرفی هر سرویس را گزارش می‌کند.
- تمام اجزا روی Linux و Kubernetes واقعی اجرا می‌شوند، نه صرفاً روی `kind`.

### کاربران و دسترسی‌ها

- `platform-admins`: مدیریت پلتفرم و زیرساخت.
- `developers`: ساخت سرویس و مشاهده و مدیریت منابع مجاز خود.
- `viewers`: مشاهده سرویس‌ها و وضعیت آن‌ها، بدون مجوز تغییر یا حذف.

**معیار موفقیت:** سارا بتواند در کمتر از ۱۰ دقیقه، بدون اجرای `kubectl` و بدون درخواست از تیم DevOps، یک سرویس زنده و متصل به صف پیام بسازد.

---

## ۲. چرا این سناریو را اجرا می‌کنیم؟

هدف اصلی کاهش اصطکاک میان توسعه‌دهنده و زیرساخت است. ابزارها به‌تنهایی ارزش محدودی دارند؛ ارزش اصلی از اتصال درست آن‌ها به یک جریان کاری کامل ایجاد می‌شود.

این تمرین کمک می‌کند:

- ابزارهای متداول IDP را در کنار یکدیگر یاد بگیریم؛
- اتصال OIDC و SSO میان ابزارها را پیاده‌سازی کنیم؛
- Git را به‌عنوان منبع حقیقت (Source of Truth) در GitOps به کار ببریم؛
- ساخت سرویس را با Software Template خودکار کنیم؛
- RBAC و دسترسی حداقلی (Least Privilege) را در عمل تمرین کنیم؛
- معماری Event-driven و مفاهیمی مانند Exchange، Queue، Retry و DLQ را یاد بگیریم؛
- خطاهای واقعی شبکه، ذخیره‌سازی، کلاستر و سرویس‌ها را عیب‌یابی کنیم.

> نکته: «بدون kubectl» به معنی حذف ابزارهای مدیریتی از کار روزمره توسعه‌دهنده است؛ عملیات bootstrap، تعمیر و عیب‌یابی زیرساخت همچنان ممکن است به دسترسی مدیریتی نیاز داشته باشد.

---

## ۳. معرفی ابزارها

| ابزار | نقش در سناریو |
|---|---|
| **Keycloak** | مدیریت هویت، کاربران، گروه‌ها، نقش‌ها و ورود یکپارچه با OIDC/SSO |
| **Argo CD** | پیاده‌سازی GitOps؛ تطبیق وضعیت Kubernetes با وضعیت تعریف‌شده در Git |
| **RabbitMQ** | پیام‌رسان برای ارتباط غیرهمزمان بین سرویس‌ها |
| **Backstage** | پورتال توسعه‌دهندگان، Catalog، Software Templates و دسترسی یکپارچه به ابزارها |
| **Vault** | نگهداری امن secretها، صدور credentialهای پویا و در صورت نیاز گواهی‌های داخلی |
| **External Secrets Operator (ESO)** | همگام‌سازی secretهای Vault با Kubernetes Secret |
| **CloudNativePG** | مدیریت PostgreSQL، replication، failover و backup |
| **OpenTelemetry** | استانداردسازی تولید و انتقال trace، metric و log |
| **Tempo** | ذخیره‌سازی distributed trace |
| **Loki** | جمع‌آوری و جست‌وجوی logها |
| **Prometheus** | جمع‌آوری metricها و تأمین داده برای alerting و تحلیل rollout |
| **Grafana** | داشبورد و ارتباط میان metric، log و trace |
| **OpenCost** | برآورد و تحلیل هزینه منابع Kubernetes |
| **Linux و Kubernetes** | زیرساخت اجرایی و محیط تمرین عیب‌یابی واقعی |

---

## ۴. زیرساخت پایه

برای توسعه اولیه می‌توان از `kind` یا `k3d` و یک Git server محلی استفاده کرد؛ اما هدف نهایی این سناریو، کلاستر چندنودی واقعی است.

- **Git server:** Gitea، برای جلوگیری از وابستگی ضروری به GitHub.
- **DNS آزمایشگاهی:** دامنه‌هایی مانند `*.localtest.me` یا ورودی‌های کنترل‌شده در `/etc/hosts`.
- **Kubernetes:** چند ماشین Linux با `kubeadm`.
- **شبکه:** Cilium و MetalLB یا راهکار معادل متناسب با محیط.
- **ذخیره‌سازی:** Longhorn یا Rook-Ceph.
- **ورودی و TLS:** Ingress/Gateway API و cert-manager.

`kind` برای آزمایش سریع مناسب است؛ اما جای تمرین خرابی واقعی نود، ذخیره‌سازی و HA را نمی‌گیرد.

---

## ۵. فازهای اجرا

### فاز ۱: Keycloak

- ساخت realm به نام `company`.
- ساخت گروه‌های `platform-admins`، `developers` و `viewers`.
- ساخت کاربران آزمایشی برای هر نقش.
- ساخت OIDC client برای Argo CD، Backstage و رابط مدیریت RabbitMQ، در صورت پشتیبانی و انتخاب روش سازگار.
- تنظیم group/role claims در tokenها.

**خروجی مورد انتظار:** هویت مرکزی و نقش‌های مشخص برای کاربران.

### فاز ۲: Argo CD

- نصب Argo CD و اتصال OIDC به Keycloak.
- تعریف RBAC براساس گروه‌ها.
- محدود کردن دسترسی developerها به projectها و applicationهای مجاز.
- ساخت ساختار App of Apps یا ApplicationSet برای نصب ابزارهای پلتفرم از Git.
- فعال‌کردن automated sync و در موارد مناسب self-heal و pruning.

**خروجی مورد انتظار:** منابع Kubernetes از Git مدیریت شوند و تغییرات دستیِ خارج از Git قابل تشخیص و اصلاح باشند.

### فاز ۳: RabbitMQ

- نصب RabbitMQ با RabbitMQ Cluster Operator از طریق Argo CD.
- پیکربندی دسترسی رابط مدیریتی با روش SSO/OAuth2 سازگار با نسخه و پلاگین‌های انتخاب‌شده؛ در غیر این صورت، مدیریت دسترسی از طریق روش احراز هویت پشتیبانی‌شده.
- ساخت `order-service` به‌عنوان producer و `notification-service` به‌عنوان consumer.
- تمرین Exchange، Queue، Binding، Dead Letter Queue، Retry و acknowledgement.
- آزمایش رفتار پیام‌ها هنگام از کار افتادن consumer.

**خروجی مورد انتظار:** پیام‌ها با الگوی قابل‌اعتماد بین سرویس‌ها جابه‌جا شوند.

### فاز ۴: Backstage

- نصب Backstage و اتصال آن به Keycloak.
- استفاده از PostgreSQL واقعی؛ نه SQLite برای محیط نهایی.
- ثبت سرویس‌ها با `catalog-info.yaml`.
- افزودن پلاگین Argo CD برای مشاهده وضعیت sync و health.
- ساخت Software Template برای ایجاد repository در Gitea، ثبت Catalog و ایجاد یا فعال‌سازی Argo CD Application.
- افزودن لینک‌های Grafana، Tempo و مستندات سرویس.

**خروجی مورد انتظار:** توسعه‌دهنده از یک پورتال، سرویس بسازد و وضعیت آن را مشاهده کند.

### فازهای تکمیلی

- **فاز ۰ و ۰.۵:** آماده‌سازی Linux و ساخت Kubernetes واقعی.
- **فاز ۱.۵:** Vault، ESO و PostgreSQL؛ این لایه باید قبل از وابسته‌شدن سرویس‌ها به secret و دیتابیس آماده باشد.
- **فاز ۳.۵:** Prometheus، Loki، Tempo و OpenTelemetry.
- **فاز ۴.۵:** OpenCost، زنجیره تأمین نرم‌افزار، Kyverno و Argo Rollouts.

---

## ۶. تست نهایی (End-to-End)

1. سارا با گروه `developers` وارد Backstage می‌شود.
2. از قالب Python، سرویس `payment-service` را می‌سازد.
3. repository و manifestها در Gitea ایجاد می‌شوند.
4. Argo CD سرویس را به‌طور خودکار deploy می‌کند.
5. سرویس از secret تأمین‌شده توسط Vault/ESO استفاده می‌کند.
6. سرویس به RabbitMQ متصل می‌شود و پیام ارسال یا دریافت می‌کند.
7. وضعیت health و sync در Backstage قابل مشاهده است.
8. رضا با گروه `viewers` وارد می‌شود و می‌تواند اطلاعات را ببیند، اما مجوز ساخت، sync، تغییر یا حذف ندارد.

در نسخه کامل، این موارد هم بررسی می‌شوند:

- ایمیج اسکن و امضا شده است؛ Kyverno ایمیج فاقد امضای معتبر را رد می‌کند.
- هیچ secretای در Git قرار ندارد.
- trace پیام از producer تا consumer در Tempo دیده می‌شود و log مرتبط در Loki قابل جست‌وجو است.
- با از دست رفتن primary دیتابیس، failover انجام می‌شود و سرویس‌ها پس از وقفه احتمالی به کار ادامه می‌دهند.
- هزینه تقریبی `payment-service` در OpenCost قابل مشاهده است.
- سناریوهای Game Day بدون از دست رفتن غیرمنتظره پیام اجرا می‌شوند.

---

## ۷. ابزارهای مکمل

| دسته | ابزار | کاربرد |
|---|---|---|
| Secrets | Vault + External Secrets Operator | نگهداری و تزریق secret بدون ذخیره در Git |
| TLS | cert-manager + Vault PKI یا step-ca | صدور و تمدید خودکار certificate |
| Observability | Prometheus + Grafana + Loki | metric، داشبورد و log |
| Tracing | OpenTelemetry + Tempo | دنبال‌کردن درخواست و پیام میان سرویس‌ها |
| Policy | Kyverno | اعمال policy، الزام resource limits و کنترل image |
| Progressive delivery | Argo Rollouts | Canary deployment و rollback براساس metric |
| CI | Gitea Actions + runner یا Tekton | build، test، scan، sign و push ایمیج |
| Registry | Harbor | نگهداری image و artifact |
| Autoscaling | KEDA | مقیاس‌دادن consumer براساس طول صف RabbitMQ |
| Infrastructure as Code | Terraform + Ansible | ساخت VMها و پیکربندی سیستم‌عامل |
| Dependency updates | Renovate | به‌روزرسانی وابستگی‌ها با Pull Request |
| Runtime security | Falco | تشخیص رفتار مشکوک در runtime |
| Chaos engineering | Chaos Mesh یا LitmusChaos | آزمایش رفتار سیستم در شرایط خرابی |
| Cost | OpenCost یا Kubecost | تحلیل هزینه و شناسایی مصرف بیش‌ازحد |

---

## ۸. ترتیب اولیه اجرا

برای شروع، ابتدا Keycloak و Argo CD و RabbitMQ را به هم متصل کن و سپس Backstage را اضافه کن؛ Backstage به‌دلیل templateها و integrationها معمولاً پیچیده‌ترین بخش این زنجیره است.

پس از اثبات جریان اصلی، ESO/Vault و KEDA را اضافه کن. بااین‌حال، در پیاده‌سازی نهایی باید وابستگی‌های دیتابیس و secret را پیش از سرویس‌هایی که به آن‌ها نیاز دارند آماده کرد. ترتیب کامل در بخش ۱۳ آمده است.

---

## ۹. زیرساخت اصلی: Linux و Kubernetes واقعی

### فاز ۰: لایه Linux

#### توپولوژی پیشنهادی

- ۳ VM برای control plane.
- ۲ VM برای worker.
- در صورت امکان، یک VM جدا برای Gitea و Harbor.
- سیستم‌عامل: Ubuntu Server یا Debian.
- ساخت VMها با Multipass، Vagrant + libvirt/KVM یا Proxmox.
- ساخت و تنظیم تکرارپذیر با Terraform/Vagrant و Ansible.

منابع CPU، RAM و دیسک را براساس ظرفیت میزبان و اندازه workloadها انتخاب کن؛ برای اجرای همه اجزای این سناریو، منابع قابل‌توجهی لازم است.

#### آماده‌سازی Linux

- غیرفعال‌کردن swap طبق نیازمندی kubeadm.
- بارگذاری ماژول‌های `overlay` و `br_netfilter`.
- تنظیم `sysctl` برای forwarding و bridge networking.
- نصب و پیکربندی `containerd` با `SystemdCgroup`.
- فعال‌سازی SSH key و کاربر غیر root با `sudo`.
- تنظیم firewall با `nftables` یا `ufw` با توجه به پورت‌های موردنیاز.
- نصب به‌روزرسانی‌های امنیتی و تنظیم hardening پایه.
- جداسازی فضای `/var/lib/containerd` عندالامکان و برنامه‌ریزی ذخیره‌سازی Longhorn/Ceph.

#### ابزارهای عیب‌یابی

`journalctl`، `systemctl`، `ss`، `tcpdump`، `strace`، `iostat`، `dmesg` و `crictl`.

**چالش:** یک worker را با پرشدن دیسک یا متوقف‌کردن kubelet مختل کن. ابتدا با ابزارهای Linux علت را پیدا کن، سپس وضعیت Kubernetes را بررسی و سرویس را بازیابی کن.

### فاز ۰.۵: Kubernetes با kubeadm

- ساخت HA control plane با ۳ نود.
- فراهم‌کردن endpoint ثابت API Server با HAProxy + Keepalived یا kube-vip.
- نصب Cilium به‌عنوان CNI؛ استفاده از eBPF و جایگزینی kube-proxy را براساس تنظیمات انتخابی Cilium بررسی کن.
- نصب MetalLB برای LoadBalancer IP در شبکه محلی.
- نصب Ingress Controller یا استفاده از Gateway API.
- نصب Longhorn یا Rook-Ceph برای PersistentVolume.
- تهیه snapshot از etcd با `etcdctl` و تمرین restore روی کلاستر آزمایشی تازه.
- استفاده از Velero برای backup منابع و volumeها، با توجه به سازگاری storage.
- تمرین ارتقای minor version، نودبه‌نود با drain و uncordon.
- تنظیم RBAC، Pod Security Admission در حالت `restricted` و NetworkPolicy با رویکرد default-deny.

**چالش‌ها:**

1. گواهی‌های کلاستر را در محیط آزمایشگاهی منقضی‌شده فرض کن و فرایند بررسی و تمدید را تمرین کن.
2. از snapshot، etcd را روی محیط جدا بازیابی کن.
3. یک worker را drain و سپس به کلاستر برگردان.
4. دسترسی شبکه بین دو namespace را با NetworkPolicy محدود و نتیجه را بررسی کن.

> عملیات خرابی و restore را ابتدا در محیط آزمایشگاهی انجام بده؛ به کلاستر یا داده‌های production دست نزن.

---

## ۱۰. فهرست کامل ابزارها

| دسته | ابزار | خروجی مورد انتظار |
|---|---|---|
| Provisioning | Terraform + Ansible | ساخت VM و تنظیم تکرارپذیر سیستم‌عامل |
| Database | CloudNativePG | PostgreSQL با replication، failover و backup |
| Secrets | Vault + ESO | حذف secretهای ثابت از Git |
| PKI | cert-manager + Vault PKI یا step-ca | گواهی داخلی و تمدید خودکار |
| Supply chain | Harbor + Trivy + Cosign | اسکن و امضای image |
| Policy | Kyverno | الزام امضا، label و resource limit |
| Runtime security | Falco | تشخیص رفتار غیرعادی container |
| CI | Gitea Actions + runner یا Tekton | build، test، scan، sign و push |
| Progressive delivery | Argo Rollouts | Canary و rollback خودکار |
| Autoscaling | KEDA | scale کردن consumer با طول صف |
| Metrics | kube-prometheus-stack | metric و dashboard کلاستر و برنامه‌ها |
| Logs | Loki + Grafana Alloy | جمع‌آوری و جست‌وجوی متمرکز log |
| Tracing | OpenTelemetry + Tempo | trace انتها‌به‌انتها |
| Alerting | Alertmanager | هشدار رشد صف، DLQ یا افزایش خطا |
| Chaos | Chaos Mesh یا LitmusChaos | تمرین خرابی کنترل‌شده |
| Cost | OpenCost یا Kubecost | گزارش مصرف منابع و هزینه تخمینی |
| Dependency updates | Renovate | PR برای آپدیت وابستگی‌ها و imageها |

---

## ۱۱. چالش‌های هر ابزار

### Keycloak

- اجرای دو replica با PostgreSQL مدیریت‌شده توسط CloudNativePG.
- حذف آزمایشی یک Pod و بررسی تداوم login.
- نگاشت گروه‌های Keycloak به نقش‌های Argo CD، Backstage و RabbitMQ.
- decode کردن token و بررسی claimهای `groups` و roleها.
- بررسی انقضا، audience، redirect URI و تنظیمات OIDC.

### Argo CD

- ثبت یک کلاستر دوم یا محیط staging.
- استفاده از ApplicationSet برای deploy یک برنامه در چند محیط.
- کنترل ترتیب نصب CRD، Operator و instance با sync waves.
- تغییر دستی یک resource و مشاهده رفتار self-heal.
- آزمایش rollback از طریق Git؛ تغییرات را در Git اصلاح کن، نه با تغییرات دستی پایدار.

### RabbitMQ

- استفاده از Quorum Queue با ۳ عضو در محیطی که منابع کافی دارد.
- از کار انداختن یک نود و بررسی پایداری پیام‌ها.
- فعال‌کردن metrics و TLS.
- ساخت کاربر اختصاصی هر سرویس و محدودکردن دسترسی به vhost و permissionهای لازم.
- تمرین publisher confirms، manual consumer acknowledgements، idempotency و poison message با DLQ.
- تنظیم KEDA براساس `queueLength` و آزمایش scale با RabbitMQ PerfTest.

### Backstage

- نصب با Helm یا روش استقرار انتخاب‌شده و اتصال به PostgreSQL.
- ساخت template چندمرحله‌ای برای repository، Catalog و Argo CD Application.
- افزودن پلاگین‌های Kubernetes، Argo CD و در صورت نیاز Prometheus.
- فعال‌کردن TechDocs برای تولید مستندات از پوشه `docs/` هر repository.

---

## ۱۲. سناریوی End-to-End و Game Day

### جریان موفق

1. سارا با نقش `developers` وارد Backstage می‌شود.
2. `payment-service` را از template می‌سازد.
3. Gitea Actions یا Tekton، image را build، test و با Trivy اسکن می‌کند.
4. image با Cosign امضا و در Harbor push می‌شود.
5. Kyverno امضای معتبر را بررسی می‌کند و image مجاز را می‌پذیرد.
6. آزمایش منفی انجام می‌شود: image بدون امضای معتبر باید رد شود.
7. Argo CD سرویس را deploy می‌کند؛ Argo Rollouts در صورت نیاز Canary را اجرا می‌کند.
8. اگر error rate از حد تعیین‌شده عبور کند، تحلیل rollout باید rollback را فعال کند.
9. سرویس secretها را از Vault از طریق ESO می‌گیرد و با کاربر اختصاصی به RabbitMQ وصل می‌شود.
10. یک پیام در Grafana/Tempo از producer تا consumer دنبال می‌شود.
11. log متناظر در Loki پیدا می‌شود.
12. رضا با نقش `viewers` اطلاعات را می‌بیند، اما نمی‌تواند منابع را تغییر دهد یا حذف کند.

### Game Day

هر خرابی را جداگانه و در محیط آزمایشگاهی اجرا کن. برای هر مورد، زمان تشخیص، اثر بر کاربر، علت ریشه‌ای، راهکار بازیابی و اقدام پیشگیرانه را در یک postmortem کوتاه ثبت کن.

- خاموش‌شدن ناگهانی یک worker.
- از دست رفتن یک control plane و بررسی پاسخ‌گویی API.
- بازیابی etcd از snapshot.
- از کار افتادن DNS داخلی (CoreDNS).
- پرشدن دیسک نودی که RabbitMQ روی آن اجرا می‌شود.
- sealed شدن Vault و بررسی اثر آن بر Podهای موجود و جدید.
- یک commit معیوب در Git و بازیابی نسخه سالم از طریق Git.
- قطع‌شدن موقت دسترسی به storage یا شبکه.
- از دست رفتن consumer هنگام وجود پیام‌های تأییدنشده.

**معیار پایان:** سرویس بازیابی شود، پیام‌ها طبق تضمین‌های طراحی‌شده مدیریت شوند و علت خرابی مستند شده باشد.

---

## ۱۳. ترتیب نهایی اجرا

ترتیب پیشنهادی با توجه به وابستگی‌ها:

1. **فاز ۰ — Linux:** ساخت VMها، شبکه، storage و hardening پایه.
2. **فاز ۰.۵ — Kubernetes:** kubeadm HA، Cilium، MetalLB، storage، backup و RBAC.
3. **زیرساخت پایه پلتفرم:** Git server، cert-manager و DNS/Ingress.
4. **PostgreSQL و Secrets:** CloudNativePG، backup، Vault و ESO.
5. **Keycloak:** realm، کاربران، گروه‌ها و OIDC.
6. **Argo CD:** اتصال OIDC، RBAC و App of Apps/ApplicationSet.
7. **RabbitMQ:** Operator، quorum queue، permissionها و سرویس‌های نمونه.
8. **Observability:** Prometheus، Grafana، Loki، Tempo و OpenTelemetry.
9. **Backstage:** Catalog، SSO، Software Template و پلاگین Argo CD.
10. **Supply chain و Policy:** Harbor، Trivy، Cosign و Kyverno.
11. **Progressive delivery و autoscaling:** Argo Rollouts و KEDA.
12. **Cost و hygiene:** OpenCost، Renovate و پیشنهادهای VPA.
13. **Game Day:** اجرای سناریوهای خرابی، بازیابی و postmortem.

اگر هدف آمادگی گواهی باشد، بخش kubeadm، etcd backup/restore، upgrade، RBAC، NetworkPolicy و سخت‌سازی Kubernetes تمرین ارزشمندی برای مسیر CKA/CKS است؛ البته این سناریو به‌تنهایی جایگزین سرفصل رسمی آزمون نیست.

---

## ۱۴. OpenTelemetry: ردگیری کامل یک پیام

### هدف

یک پیام را در یک trace واحد از `order-service`، از طریق RabbitMQ، تا `notification-service` دنبال کن. این کار برای پیدا کردن تأخیر و خطا در معماری پیام‌محور ضروری است.

### اجزا

- OpenTelemetry Operator برای مدیریت Collector و قابلیت‌های instrumentation.
- Collector به‌شکل DaemonSet برای دریافت داده‌های محلی و/یا Deployment برای پردازش و ارسال داده‌ها، متناسب با معماری.
- Tempo برای نگهداری traceها.
- Prometheus برای metricها.
- Loki برای logها.
- Grafana برای ارتباط میان trace، log و metric.

### مراحل

1. Operator و Collector را از طریق Argo CD نصب کن.
2. خروجی traceها را به Tempo ارسال کن.
3. در `order-service` و `notification-service`، OpenTelemetry SDK را اضافه کن.
4. هنگام publish پیام RabbitMQ، context مربوط به trace را با header استاندارد W3C `traceparent` منتقل کن.
5. consumer آن header را بخواند و span جدید را در همان trace ادامه دهد.
6. logها را ساختارمند تولید کن و `trace_id` را در آن‌ها قرار بده.
7. لینک Grafana و Tempo را در صفحه سرویس Backstage اضافه کن.
8. instrumentation را در template سرویس‌های جدید فعال کن.

### چالش‌ها

- در consumer تأخیر عمدی ایجاد کن و span کند را پیدا کن.
- پیام را به DLQ بفرست و رفتار trace را هنگام retry بررسی کن.
- Sampling را برای staging روی ۱۰۰٪ تنظیم کن و برای production براساس حجم و نیاز، sampling مناسب انتخاب کن.
- از یک trace به log مرتبط از طریق `trace_id` برو.
- برای تأخیر بین publish و consume بیشتر از ۵ ثانیه هشدار تعریف کن.

---

## ۱۵. ابزارهای تکمیلی: Vault، Loki، PostgreSQL و Cost

### ۱۵.۱ Vault: مدیریت Secret

Vault secretها را به‌شکل رمزنگاری‌شده نگه می‌دارد و policy مشخص می‌کند هر هویت به کدام secret دسترسی دارد.

#### مراحل

1. Vault را با Helm نصب کن.
2. برای HA از storage و معماری پشتیبانی‌شده، مانند integrated Raft با تعداد مناسب replica، استفاده کن.
3. Kubernetes auth را فعال کن.
4. برای هر ServiceAccount policy جداگانه و دسترسی حداقلی تعریف کن.
5. ESO را به Vault متصل کن و برای credentialهای RabbitMQ و سایر برنامه‌ها `ExternalSecret` بساز.
6. در صورت نیاز، Vault PKI را با cert-manager برای certificateهای داخلی یکپارچه کن.
7. audit log را فعال کن و دسترسی به secretها را بررسی کن.

#### چالش‌ها

- Vault را در محیط آزمایشگاهی seal کن؛ اثر آن را بر دسترسی Podهای موجود و ساخت Pod جدید بررسی کن، سپس سرویس را امن بازیابی کن.
- برای اتصال دیتابیس، dynamic secret با TTL یک‌ساعته بساز و انقضا و تمدید آن را بررسی کن.

> Vault به‌طور معمول secretهای از قبل تزریق‌شده را از حافظه Pod حذف نمی‌کند. دسترسی برنامه‌های جدید یا refresh شدن secretها ممکن است تا بازگشت Vault مختل شود؛ این رفتار را در آزمایش ثبت کن.

### ۱۵.۲ Loki: لاگ مرکزی

#### مراحل

1. Loki را با معماری متناسب با حجم داده نصب کن.
2. برای ذخیره‌سازی پایدار، از object storage سازگار با S3 مانند MinIO استفاده کن، در صورت انتخاب معماری‌ای که به آن نیاز دارد.
3. Grafana Alloy را به‌صورت DaemonSet اجرا کن تا logهای نود و Podها را جمع‌آوری کند.
4. logهای برنامه را ساختارمند و شامل `trace_id` تولید کن.
5. retention متناسب با محیط تعیین کن؛ برای نمونه، ۷ روز در dev و ۳۰ روز در production، مشروط به ظرفیت و سیاست نگهداری.

#### نمونه کوئری‌های LogQL

خطاهای `payment-service` در پنج دقیقه اخیر:

```logql
{app="payment-service"} |= "ERROR"
```

فقط لاگ‌های دارای شناسه trace:

```logql
{app="payment-service"} | json | trace_id != ""
```

برای استفاده از این کوئری‌ها، labelها و قالب واقعی log باید با آنچه برنامه تولید می‌کند تطبیق داده شوند.

#### چالش‌ها

- هشدار برای بیش از ۱۰ خطا در دقیقه بساز.
- حجم log را بررسی کن و namespaceهای پرمصرف را شناسایی کن.
- مطمئن شو labelهای با cardinality بالا مانند شناسه‌های منحصربه‌فرد درخواست را بی‌رویه به‌عنوان Loki label استفاده نمی‌کنی.

### ۱۵.۳ PostgreSQL: دیتابیس Keycloak و Backstage

#### مراحل

1. CloudNativePG Operator را از طریق Argo CD نصب کن.
2. یک Cluster با سه instance بساز: یک primary و دو replica.
3. دیتابیس‌های `keycloak` و `backstage` را جدا بساز.
4. برای هر برنامه user اختصاصی ایجاد کن.
5. credentialها را از Vault و ESO تأمین کن.
6. backup به MinIO یا S3، WAL archiving و Point-in-Time Recovery (PITR) را پیکربندی و آزمایش کن.
7. در صورت نیاز، connection pooling را با PgBouncer یا قابلیت سازگار و پشتیبانی‌شده در نسخه انتخابی CloudNativePG فعال کن.
8. metricهای PostgreSQL را به Prometheus بده و dashboard بساز.

#### چالش‌ها

- Pod مربوط به primary را در محیط آزمایشگاهی حذف کن و زمان failover و اثر آن بر Keycloak را اندازه بگیر.
- در دیتابیس آزمایشی، حذف کنترل‌شده‌ای انجام بده و با PITR به زمان قبل از حذف برگرد.
- rolling update نسخه PostgreSQL را طبق روش پشتیبانی‌شده تمرین کن.

### ۱۵.۴ Cost: هزینه منابع

OpenCost هزینه تخمینی CPU، RAM و سایر منابع قابل پشتیبانی را نشان می‌دهد. در کلاستر آزمایشگاهی هم می‌توان قیمت فرضی تعریف کرد تا اصول right-sizing تمرین شوند.

#### مراحل

1. OpenCost را نصب و اتصال آن به Prometheus را تنظیم کن.
2. نرخ فرضی برای vCPU و هر GB RAM تعریف کن.
3. labelهای `team` و `app` را روی workloadها اعمال کن.
4. گزارش هزینه namespaceهایی مانند `keycloak`، `rabbitmq`، `backstage` و `payment-service` را بررسی کن.
5. مصرف واقعی را با CPU/memory requests مقایسه کن.
6. VPA را در حالت توصیه‌گر (recommendation-only) برای پیشنهاد request مناسب به کار ببر.
7. در صورت نیاز، با Kyverno الزام وجود labelها را اعمال کن.

#### چالش‌ها

- requestهای `payment-service` را براساس پیشنهاد VPA و با احتیاط تنظیم کن و گزارش هزینه را دوباره بررسی کن.
- policyای بنویس که Deployment فاقد label `team` را رد کند.
- مشخص کن کدام هزینه‌ها تخمینی هستند و کدام هزینه‌های واقعی زیرساخت خارج از محاسبات OpenCost قرار دارند.

---

## تعریف پایان پروژه

پروژه زمانی کامل است که:

- [ ] زیرساخت Linux و Kubernetes قابل بازسازی و مستند باشد.
- [ ] بیشتر منابع پلتفرم پس از bootstrap از Git مدیریت شوند.
- [ ] ورود یکپارچه و نقش‌های `platform-admins`، `developers` و `viewers` تست شده باشند.
- [ ] Backstage بتواند سرویس جدید را از template ایجاد کند.
- [ ] Argo CD سرویس را از Git deploy کند.
- [ ] secret در Git وجود نداشته باشد و از مسیر Vault/ESO تأمین شود.
- [ ] producer و consumer از RabbitMQ استفاده کنند و رفتار failure آزمایش شده باشد.
- [ ] log، metric و trace در ابزارهای مربوط قابل مشاهده باشند.
- [ ] policy امضای image و الزام labelها تست شده باشد.
- [ ] گزارش هزینه تقریبی سرویس قابل مشاهده باشد.
- [ ] حداقل چند Game Day اجرا و برای هرکدام postmortem نوشته شده باشد.

## نکات عملیاتی

- نسخه ابزارها، chartها و Operatorها را pin کن و تغییر نسخه را از طریق Pull Request انجام بده.
- credential، token، private key و kubeconfig را در Git commit نکن.
- برای هر سرویس ServiceAccount و مجوزهای حداقلی بساز.
- backup فقط وقتی قابل اتکاست که restore آن عملاً آزمایش شده باشد.
- HA به‌تنهایی جایگزین backup، مانیتورینگ و تمرین بازیابی نیست.
- ابتدا مسیر ساده‌ی «ساخت سرویس → deploy → پیام RabbitMQ» را کامل کن؛ سپس ابزارهای امنیت، مشاهده‌پذیری، هزینه و chaos را مرحله‌ای اضافه کن.
"""
path = Path("/mnt/data/IDP-Scenario-README.md")
path.write_text(readme, encoding="utf-8")
print(f"Created: {path}\nSize: {path.stat().st_size:,} bytes")

