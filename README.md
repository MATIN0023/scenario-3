سناریوی تمرینی: ساخت Internal Developer Platform (IDP)
⁧ابزارها: Keycloak، ArgoCD، RabbitMQ، Backstage، Vault، PostgreSQL، OpenTelemetry، Loki، OpenCost، روی Linux و Kubernetes واقعی⁩
۱. سناریوی کاربر (User Scenario)
کاربر: سارا، developer جدید در شرکت فرضی «کمپانی».
وضعیت فعلی (قبل از پلتفرم): سارا باید سرویس جدیدی به اسم payment-service بسازد. برای این کار باید از تیم DevOps بخواهد repo بسازد، pipeline تنظیم کند، manifestهای Kubernetes را بنویسد، دسترسی به RabbitMQ بگیرد و برای هر ابزار یک username و password جدا داشته باشد. این کار معمولاً چند روز طول می‌کشد.
وضعیت هدف (بعد از پلتفرم):
سارا فقط یک بار با حساب خودش در Keycloak وارد می‌شود و به Backstage، ArgoCD و RabbitMQ دسترسی دارد (SSO).
در Backstage روی دکمه «Create» می‌زند و template «Python Service» را انتخاب می‌کند.
نام سرویس (payment-service) را می‌نویسد و تأیید می‌کند.
Backstage خودکار یک repo جدید با کد اسکلت و manifestها در Git می‌سازد و سرویس را در Catalog ثبت می‌کند.
ArgoCD تغییر را در Git می‌بیند و سرویس را بدون هیچ دستور دستی deploy می‌کند.
سرویس به RabbitMQ وصل می‌شود و پیام می‌فرستد و می‌گیرد.
سارا وضعیت سلامت و sync سرویس را در همان صفحه Backstage می‌بیند.
پشت صحنه (چیزهایی که سارا نمی‌بیند ولی همیشه کار می‌کنند):
پسورد RabbitMQ را Vault و External Secrets به pod می‌رسانند؛ هیچ secret در Git نیست.
Keycloak و Backstage روی PostgreSQL با CloudNativePG اجرا می‌شوند (۳ instance و backup خودکار).
هر پیام یک trace با OpenTelemetry دارد که در Tempo و Grafana دیده می‌شود و لاگ‌هایش در Loki است.
هزینه منابع هر سرویس در OpenCost دیده می‌شود.
همه‌چیز روی کلاستر واقعی Kubernetes و چند ماشین Linux اجرا می‌شود، نه روی kind.
کاربر دوم: رضا، با نقش viewers. او می‌تواند سرویس‌ها و وضعیتشان را ببیند، اما نمی‌تواند سرویس بسازد، sync کند یا queue حذف کند.
معیار موفقیت: سارا در کمتر از ۱۰ دقیقه، بدون درخواست از تیم DevOps و بدون kubectl، یک سرویس زنده و متصل به صف پیام دارد.
۲. چرا این سناریو را می‌خواهیم؟
دلیل اصلی این است که این چهار ابزار به‌تنهایی کاربرد محدودی دارند، ولی کنار هم یک مسئله واقعی را حل می‌کنند: کاهش اصطکاک بین developer و زیرساخت.
تمرین روی مسئله واقعی: در شرکت‌ها دقیقاً همین ترکیب (Backstage + ArgoCD + Keycloak) برای ساخت پلتفرم داخلی استفاده می‌شود. یادگیری کنار هم، از یادگیری جداگانه ارزشمندتر است.
فهم ارتباط ابزارها: سخت‌ترین بخش DevOps نصب ابزار نیست، اتصال آن‌هاست: OIDC بین Keycloak و بقیه، Git به‌عنوان منبع حقیقت برای ArgoCD و template در Backstage.
تمرین GitOps واقعی: بعد از bootstrap اولیه، هیچ kubectl apply دستی نداریم و هر تغییر از Git می‌گذرد.
تمرین امنیت و دسترسی: نقش‌های platform-admins، developers و viewers نشان می‌دهند RBAC در عمل چطور کار می‌کند.
تمرین معماری event-driven: RabbitMQ با سناریوی ساده order و notification، مفاهیم exchange، queue، DLQ و retry را ملموس می‌کند.
۳. معرفی ابزارها
Keycloak (هویت و SSO)
سرور مدیریت هویت متن‌باز. کاربران، گروه‌ها و نقش‌ها را یک‌جا نگه می‌دارد و با OIDC به بقیه ابزارها login می‌دهد. نقش در سناریو: یک login برای همه، و تعیین نقش هر کاربر.
ArgoCD (GitOps)
ابزار deploy برای Kubernetes که وضعیت کلاستر را با آنچه در Git نوشته شده مقایسه و همگام می‌کند. نقش در سناریو: هر سرویس جدید که در Git ثبت شود، خودکار deploy می‌شود. خود Keycloak و RabbitMQ و Backstage هم از همین راه نصب می‌شوند (App of Apps / ApplicationSet).
RabbitMQ (پیام‌رسان)
message broker برای ارتباط غیرهمزمان بین سرویس‌ها. با Cluster Operator روی Kubernetes نصب می‌شود. نقش در سناریو: order-service پیام می‌فرستد و notification-service آن را مصرف می‌کند.
Backstage (پورتال توسعه‌دهنده)
پلتفرم پورتال داخلی از Spotify. سه کار اصلی دارد: Catalog سرویس‌ها، Software Template برای ساخت سرویس جدید، و پلاگین‌ها (مثل ArgoCD). نقش در سناریو: نقطه ورود developer؛ جایی که سرویس می‌سازد و وضعیتش را می‌بیند.
Vault (مدیریت Secret)
سرور نگهداری رمزنگاری‌شده پسوردها و توکن‌ها. نقش در سناریو: هیچ پسوردی در Git نیست؛ سرویس‌ها secret خود را از Vault و External Secrets می‌گیرند.
PostgreSQL با CloudNativePG (دیتابیس)
دیتابیس رابطه‌ای HA روی Kubernetes با Operator. نقش در سناریو: دیتابیس واقعی برای Keycloak و Backstage، با failover و backup.
OpenTelemetry، Loki و Tempo (مشاهده‌پذیری)
OpenTelemetry ردگیری (trace) تولید می‌کند، Tempo آن را نگه می‌دارد و Loki لاگ‌ها را جمع می‌کند؛ همه در Grafana دیده می‌شوند. نقش در سناریو: ردگیری یک پیام از order-service تا notification-service و پیدا کردن خطا از روی لاگ.
OpenCost (هزینه منابع)
گزارش مصرف و هزینه CPU، RAM و دیسک هر سرویس و namespace. نقش در سناریو: درست‌سایز کردن منابع و دیدن هزینه هر سرویس.
Linux و Kubernetes (زیربنا)
چند ماشین لینوکسی با kubeadm و Cilium، MetalLB و Longhorn. نقش در سناریو: مشکلات واقعی شبکه، دیسک و upgrade را تجربه می‌کنی، نه یک کلاستر ساده و بی‌مشکل.
۴. زیرساخت پایه
کلاستر لوکال: kind یا k3d همراه با ingress-nginx
Git سرور لوکال: Gitea (بدون وابستگی به GitHub)
دامنه‌های لوکال با *.localtest.me یا /etc/hosts
در سناریوی اصلی، به‌جای kind از کلاستر واقعی استفاده می‌شود: چند ماشین لینوکسی با kubeadm (فاز ۰ و ۰.۵ در بخش ۹).
۵. فازهای اجرا
فاز ۱: Keycloak
ساخت realm به اسم company
ساخت گروه‌های platform-admins، developers، viewers و چند کاربر تستی
ساخت سه client برای ArgoCD، Backstage و RabbitMQ Management
هدف: با یک login، هر سه ابزار باز شود.
فاز ۲: ArgoCD
نصب و اتصال OIDC به Keycloak
RBAC بر اساس گروه: platform-admins ادمین، developers فقط پروژه‌های خودشان
ساختار App of Apps یا ApplicationSet تا بقیه ابزارها از Git deploy شوند
فاز ۳: RabbitMQ
نصب با RabbitMQ Cluster Operator از طریق ArgoCD
اتصال management UI به Keycloak با OAuth2
نوشتن order-service و notification-service
تمرین: exchange، queue، binding، Dead Letter Queue، retry، و تست اینکه با kill کردن consumer پیام‌ها گم نمی‌شوند
فاز ۴: Backstage
نصب و login با Keycloak
ثبت سرویس‌ها با catalog-info.yaml
پلاگین ArgoCD برای دیدن وضعیت sync
Software Template که repo جدید در Gitea بسازد و ArgoCD Application را خودکار ثبت کند
فازهای تکمیلی (جزو سناریوی اصلی)
فاز ۰ و ۰.۵: Linux و Kubernetes واقعی، قبل از فاز ۱ (بخش ۹)
فاز ۱.۵: Vault، External Secrets و PostgreSQL؛ قبل از Keycloak و Backstage چون به دیتابیس و secret نیاز دارند (بخش ۱۵)
فاز ۳.۵: مشاهده‌پذیری با Loki، OpenTelemetry، Tempo و Prometheus (بخش ۱۴ و ۱۵)
فاز ۴.۵: هزینه با OpenCost، زنجیره تأمین (Harbor، Trivy، Cosign، Kyverno) و Argo Rollouts (بخش ۱۰ و ۱۵)
۶. تست نهایی (End-to-End)
ورود به Backstage با کاربر developers
ساخت payment-service از template
پیدا کردن و deploy خودکار توسط ArgoCD
اتصال سرویس به RabbitMQ و ارسال پیام
دیدن وضعیت سلامت در Backstage
ورود با کاربر viewers و بررسی اینکه دسترسی نوشتن ندارد
در نسخه کامل، این موارد هم باید تأیید شوند:
ایمیج امضاشده و اسکن‌شده است و Kyverno ایمیج بدون امضا را رد می‌کند.
پسورد از Vault می‌آید و در Git نیست.
یک پیام در Tempo به‌صورت یک trace از order-service تا notification-service دیده می‌شود و لاگ‌اش در Loki است.
با kill کردن primary در PostgreSQL، Keycloak پس از چند ثانیه دوباره کار می‌کند.
هزینه payment-service در OpenCost دیده می‌شود.
سناریوهای Game Day (بخش ۱۲) بدون از دست رفتن پیام تمام می‌شوند.
۷. ابزارهای مکمل دیگر
دسته
ابزار
کاربرد در سناریو
Secrets
External Secrets Operator + Vault
پسوردها در Git نباشند
TLS
cert-manager
certificate خودکار برای ingressها
Observability
Prometheus + Grafana + Loki
مانیتور صف‌ها و لاگ‌ها
Tracing
OpenTelemetry
ردگیری پیام از producer تا consumer
Policy
Kyverno
اجبار resource limit روی Deploymentها
Progressive delivery
Argo Rollouts
canary deployment
CI
Gitea Actions / Tekton + Harbor
build و push ایمیج
Autoscaling
KEDA
scale شدن consumer بر اساس طول صف RabbitMQ
Infra as code
Crossplane
provision منابع از داخل کلاستر
Dependency update
Renovate
آپدیت خودکار chart و ایمیج با PR

۸. ترتیب اولیه (فازهای ۱ تا ۴)
اول فاز ۱ تا ۳ (Keycloak، ArgoCD، RabbitMQ)، بعد Backstage چون سخت‌ترین است. سپس KEDA و External Secrets که بیشترین ارزش آموزشی را دارند.
ترتیب کامل و نهایی با همه ابزارها در بخش ۱۳ آمده است.
۹. زیرساخت اصلی: Linux و Kubernetes واقعی
این بخش جزو سناریوی اصلی است و «زیرساخت پایه» (بخش ۴) را کامل می‌کند. به‌جای kind که همه‌چیز را قایم می‌کند، کلاستر واقعی روی چند ماشین Linux می‌سازی تا مشکلات واقعی را ببینی. سناریوی کاربر (سارا و رضا) همان می‌ماند؛ فقط زیرش سخت‌تر می‌شود.
فاز ۰: لایه Linux
چهار تا پنج VM بساز (Multipass، Vagrant + libvirt/KVM یا Proxmox) با Ubuntu Server یا Debian:
۳ نود control-plane، ۲ نود worker، و در صورت امکان یک VM جدا برای Gitea و Harbor
ساخت VMها با Terraform یا Vagrantfile، و تنظیم آن‌ها با Ansible (همه‌چیز تکرارپذیر؛ هیچ تنظیم دستی)
آماده‌سازی نود برای Kubernetes: غیرفعال کردن swap، ماژول‌های overlay و br_netfilter، تنظیم sysctl برای forwarding، نصب containerd با SystemdCgroup
Hardening پایه: SSH فقط با key، کاربر غیر root با sudo، فایروال nftables یا ufw، fail2ban، به‌روزرسانی خودکار امنیتی
ذخیره‌سازی: دیسک جدا با LVM برای /var/lib/containerd و دیسک خام برای Longhorn یا Ceph
ابزارهای عیب‌یابی که باید یاد بگیری: journalctl، systemctl، ss، tcpdump، strace، iostat، dmesg، crictl
چالش: یک نود را عمداً با پر کردن دیسک یا kill کردن kubelet خراب کن و فقط با ابزارهای Linux علتش را پیدا کن.
فاز ۰.۵: Kubernetes واقعی با kubeadm
نصب کلاستر HA با kubeadm و سه control-plane، با HAProxy + keepalived (یا kube-vip) برای VIP سرور API
CNI: Cilium (با eBPF و جایگزینی kube-proxy) به‌جای Flannel ساده
LoadBalancer: MetalLB برای IP خارجی روی شبکه لوکال
Ingress: ingress-nginx یا Gateway API (با Cilium Gateway)
Storage: Longhorn یا Rook-Ceph برای PersistentVolume
Backup: etcdctl snapshot save و restore کامل روی کلاستر تازه؛ همچنین Velero برای backup منابع و volumeها
Upgrade: ارتقای کلاستر از یک minor version به نسخه بعدی (kubeadm upgrade)، نود به نود با drain و uncordon
امنیت: RBAC دقیق، Pod Security Admission در حالت restricted، NetworkPolicy با رویکرد default-deny
چالش: گواهی‌های کلاستر را منقضی‌شده فرض کن و با kubeadm certs renew درستشان کن.
۱۰. فهرست کامل ابزارهای سناریو
دسته
ابزار
چه چیزی به سناریو اضافه می‌کند
Provisioning
Terraform + Ansible
ساخت و تنظیم VMها به‌صورت کد
Database
CloudNativePG (PostgreSQL Operator)
دیتابیس HA برای Keycloak و Backstage، با backup خودکار
Secrets
Vault + External Secrets Operator
هیچ secret در Git نیست؛ Vault با Kubernetes auth
PKI
cert-manager + Vault PKI یا step-ca
CA داخلی و TLS خودکار بین همه سرویس‌ها
Supply chain
Harbor + Trivy + Cosign
اسکن آسیب‌پذیری و امضای ایمیج
Policy
Kyverno
فقط ایمیج امضاشده اجرا شود؛ resource limit اجباری باشد؛ latest ممنوع
Runtime security
Falco
تشخیص رفتار مشکوک در کانتینر (مثلاً باز شدن shell)
CI
Gitea Actions + runner، یا Tekton
build، تست، اسکن، امضا و push خودکار
Progressive delivery
Argo Rollouts
canary با تحلیل خودکار از Prometheus و rollback
Autoscaling
KEDA
scale شدن consumer بر اساس طول صف RabbitMQ
Metrics
kube-prometheus-stack
داشبورد Grafana برای نودها، RabbitMQ و سرویس‌ها
Logs
Loki + Promtail (یا Alloy)
لاگ مرکزی قابل جست‌وجو
Tracing
OpenTelemetry + Tempo
ردگیری یک پیام از order-service تا notification-service
Alerting
Alertmanager
هشدار وقتی صف رشد می‌کند یا DLQ پر می‌شود
Chaos
Chaos Mesh یا LitmusChaos
خراب کردن عمدی pod و شبکه و نود
Cost / hygiene
Renovate + Kubecost (اختیاری)
آپدیت خودکار وابستگی‌ها و دید هزینه منابع

۱۱. چالش‌های هر ابزار
Keycloak
اجرا با ۲ replica و دیتابیس CloudNativePG؛ یک pod را بکش و ببین login قطع می‌شود یا نه.
Role mapping: گروه‌های Keycloak را به نقش‌های ArgoCD، RabbitMQ (tag administrator) و Backstage نگاشت کن.
Token را با jwt.io-مانند ابزار decode کن و claim groups را بررسی کن.
ArgoCD
چند کلاستر: یک کلاستر دوم (یا namespace جدا) را به‌عنوان staging ثبت کن و با ApplicationSet فقط با یک فایل به هر دو deploy کن.
Sync waves و hooks: ترتیب نصب CRD، Operator، بعد instance را کنترل کن.
Drift: یک resource را دستی تغییر بده و ببین selfHeal برش می‌گرداند.
RabbitMQ
Quorum Queue روی ۳ replica به‌جای classic queue؛ یک نود را بکش و ببین پیام‌ها می‌مانند.
Prometheus metrics، TLS بین client و broker، و کاربر جدا برای هر سرویس با دسترسی حداقلی (vhost + permission).
Publisher confirms، consumer ack دستی، idempotency (یک پیام دوبار نرسد اثر دوبار نگذارد)، و Poison message با DLQ.
KEDA را با queueLength تنظیم کن و با load test (مثلاً PerfTest خود RabbitMQ) scale شدن را ببین.
Backstage
نصب با Helm و دیتابیس PostgreSQL واقعی (نه SQLite).
Template سه‌مرحله‌ای: ساخت repo در Gitea، ثبت در Catalog، و ساخت ArgoCD Application.
پلاگین‌های Kubernetes و Prometheus برای دیدن pod و متریک در صفحه سرویس.
TechDocs: مستندات هر سرویس از docs/ در repo خودش ساخته شود.
۱۲. سناریوی End-to-End نهایی و Game Day
سارا (developers) وارد Backstage می‌شود و payment-service را از template می‌سازد.
Pipeline در Gitea Actions ایمیج را build، با Trivy اسکن، با Cosign امضا و در Harbor push می‌کند.
Kyverno ایمیج امضاشده را قبول می‌کند؛ ایمیج بدون امضا رد می‌شود (این را عمداً تست کن).
ArgoCD سرویس را با canary (Argo Rollouts) deploy می‌کند و اگر error rate در Prometheus بالا رفت، خودش rollback می‌کند.
سرویس secretها را از Vault (از طریق External Secrets) می‌گیرد و با کاربر اختصاصی به RabbitMQ وصل می‌شود.
در Grafana و Tempo یک پیام را از ابتدا تا انتها دنبال می‌کنی.
رضا (viewers) وارد می‌شود و هیچ دسترسی نوشتن ندارد؛ هر تلاش او در لاگ ثبت می‌شود.
سناریوهای خرابی (Game Day)
بعد از کار کردن همه‌چیز، با Chaos Mesh یا دستی این‌ها را امتحان کن و برای هرکدام یک postmortem کوتاه بنویس:
یک نود worker ناگهان خاموش شود
یک control-plane از کار بیفتد (آیا API هنوز جواب می‌دهد؟)
etcd خراب شود و باید از snapshot restore کنی
DNS داخلی (CoreDNS) از کار بیفتد
دیسک نودی که RabbitMQ روی آن است پر شود
Vault sealed شود (secretها چه می‌شوند؟)
یک commit بد در Git باعث خراب شدن deploy شود و باید با ArgoCD rollback کنی
۱۳. ترتیب نهایی اجرا
فاز ۰ و ۰.۵: Linux و Kubernetes با kubeadm (حدود ۲ هفته؛ اینجا بیشترین یادگیری است)
Cilium، MetalLB، Longhorn، cert-manager، Vault و External Secrets
Keycloak و ArgoCD و App of Apps (از اینجا به بعد فقط Git)
RabbitMQ با Quorum Queue و سرویس‌ها و KEDA
Observability: Prometheus، Loki، Tempo
Backstage و Template
Supply chain (Harbor، Trivy، Cosign، Kyverno) و Argo Rollouts
Game Day و نوشتن postmortem
پیشنهاد: اگر هدفت گواهی است، فاز ۰.۵ تقریباً تمرین مستقیم CKA و CKS است (kubeadm، etcd backup، upgrade، NetworkPolicy، RBAC، Falco و امنیت supply chain).
۱۴. OpenTelemetry: ردگیری کامل یک پیام
هدف این بخش این است که یک سفارش را از order-service تا RabbitMQ و بعد notification-service در یک trace واحد ببینی؛ یعنی دقیقاً همان چیزی که بدون tracing در سیستم‌های پیام‌محور دیده نمی‌شود.
اجزا
OpenTelemetry Operator روی کلاستر، که Collector و auto-instrumentation را مدیریت می‌کند (نصب از طریق ArgoCD)
OTel Collector در حالت DaemonSet برای جمع‌آوری و Deployment برای پردازش و ارسال
Tempo برای نگهداری traceها، Prometheus برای متریک و Loki برای لاگ
Grafana با اتصال بین سه سیگنال (از روی یک trace به لاگ و متریک همان سرویس برو)
مراحل اجرا
Operator و Collector را با ArgoCD نصب کن و خروجی traceها را به Tempo بده.
در order-service و notification-service کتابخانه OpenTelemetry SDK را اضافه کن (Python، Go یا Node).
هنگام publish، اطلاعات trace را در header پیام RabbitMQ بگذار (traceparent، استاندارد W3C Trace Context).
در consumer، همان header را بخوان تا span جدید زیر همان trace ساخته شود.
در Backstage، لینک مستقیم به Grafana و Tempo را در صفحه سرویس بگذار.
در template سرویس جدید، instrumentation به‌صورت پیش‌فرض فعال باشد تا payment-service هم بدون کار اضافه trace داشته باشد.
چالش‌ها
یک پیام را عمداً در consumer دیر پردازش کن و در Tempo ببین کدام span کند است.
پیامی را به DLQ بفرست و ببین trace آن چطور ادامه پیدا می‌کند (retry چند span می‌سازد؟).
Sampling را تنظیم کن: ۱۰۰٪ در staging و ۱۰٪ یا tail-based sampling فقط برای خطاها در production.
از روی یک trace به لاگ همان request برو، با trace_id در لاگ‌ها (correlation).
یک هشدار بساز که اگر تأخیر بین publish و consume از ۵ ثانیه بیشتر شد فعال شود.
۱۵. چهار ابزار تکمیلی: Vault، Loki، PostgreSQL و Cost
این چهار ابزار قبلاً فقط در جدول بخش ۱۰ بودند. اینجا هرکدام نقش، مراحل نصب و چالش خودش را دارد.
۱۵.۱ Vault: مدیریت Secret
Vault تمام پسوردها و توکن‌ها را رمزنگاری‌شده نگه می‌دارد تا هیچ secret در Git نباشد.
نصب Vault با Helm در حالت HA و ذخیره‌سازی Raft (روی ۳ replica)
فعال کردن Kubernetes auth تا هر ServiceAccount فقط به secretهای خودش برسد
اتصال External Secrets Operator به Vault و ساخت ExternalSecret برای پسورد RabbitMQ و Keycloak
استفاده از Vault PKI برای صدور certificate داخلی، همراه cert-manager
فعال کردن audit log برای دیدن اینکه چه کسی کدام secret را خوانده
چالش: Vault را عمداً sealed کن و ببین سرویس‌های در حال اجرا و pod جدید چه می‌شوند؛ بعد با unseal برگردان.
چالش: برای دیتابیس، dynamic secret بساز (username موقت با TTL یک ساعت).
۱۵.۲ Loki: لاگ مرکزی
Loki لاگ همه podها را جمع می‌کند و در Grafana با زبان LogQL جست‌وجو می‌شود.
نصب Loki در حالت Simple Scalable یا Distributed، با ذخیره‌سازی روی MinIO (سازگار با S3)
جمع‌آوری لاگ با Grafana Alloy (یا Promtail) به‌صورت DaemonSet روی هر نود
لاگ ساختارمند JSON در سرویس‌ها، با فیلد trace_id برای اتصال به Tempo
Retention مثلاً ۷ روز برای dev و ۳۰ روز برای production
نوشتن چند کوئری LogQL: خطاهای payment-service در ۵ دقیقه اخیر، تعداد پیام‌های DLQ، و login ناموفق Keycloak
چالش: یک هشدار از روی لاگ بساز (مثلاً بیش از ۱۰ خطا در دقیقه) و در Alertmanager ثبتش کن.
چالش: حجم لاگ را در Grafana ببین و بفهم کدام namespace بیشترین هزینه ذخیره‌سازی را دارد.
۱۵.۳ PostgreSQL: دیتابیس واقعی
Keycloak و Backstage هر دو به دیتابیس نیاز دارند و نباید از دیتابیس داخلی یا SQLite استفاده کنند.
نصب CloudNativePG Operator از طریق ArgoCD
ساخت یک Cluster با ۳ instance (یک primary و دو replica) و دو دیتابیس جدا: keycloak و backstage
هر برنامه کاربر و پسورد اختصاصی خودش را دارد (پسورد از Vault و ESO)
Backup خودکار به MinIO یا S3 با WAL archiving و امکان بازگردانی به یک لحظه مشخص (PITR)
Connection pooling با PgBouncer که خود CloudNativePG پشتیبانی می‌کند
متریک‌ها در Prometheus و داشبورد آماده Grafana
چالش: pod primary را بکش و ببین failover چند ثانیه طول می‌کشد و Keycloak چه می‌شود.
چالش: جدولی را عمداً حذف کن و با PITR به ۵ دقیقه قبل برگردان.
چالش: نسخه PostgreSQL را با rolling update ارتقا بده.
۱۵.۴ Cost: دید هزینه منابع
منظور هزینه مصرف CPU، RAM و دیسک هر سرویس و تیم است، حتی روی کلاستر لوکال، تا عادت درست‌سایز کردن منابع شکل بگیرد.
نصب OpenCost (پروژه متن‌باز CNCF) یا Kubecost، با داده از Prometheus
قیمت فرضی برای هر vCPU و هر GB رم تعریف کن (مثلاً قیمت یک ابر معمولی)
برچسب team و app روی همه workloadها (با Kyverno اجباری شود)
گزارش هزینه هر namespace: keycloak، rabbitmq، backstage، payment-service
پیدا کردن workloadهایی که request منابع بیشتر از مصرف واقعی دارند (over-provisioned)
ابزار کمکی: VPA در حالت توصیه‌گر، برای پیشنهاد request و limit درست
چالش: منابع payment-service را بر اساس پیشنهاد VPA کم کن و اثرش را در گزارش هزینه ببین.
چالش: یک Kyverno policy بنویس که deploy بدون برچسب team را رد کند.




