# راهنمای نصب آفلاین cert-manager، CA داخلی و F5 NGINX Ingress با HTTPS

> **هدف:** مستندسازی معماری و بازتولید آزمایشگاهی TLS برای `cert.security.local` روی Kubernetes آفلاین. این مخزن **فاقد Private Key** است.
>
> **وضعیت:** جزئیات نسخه‌ها، نام منابع و سرویس‌ها از خروجی کلاستر استخراج شده‌اند؛ Helm values اصلی و YAML اولیه Root CA در اختیار این مستند نیستند. فایل‌های `manifests/` **نمونه‌های بازتولیدی** هستند و نباید بدون بررسی روی کلاستر فعلی دوباره Apply شوند.

## 1. وضعیت و معماری ثبت‌شده

| مؤلفه | پیکربندی مشاهده‌شده |
| --- | --- |
| Kubernetes | `v1.37.1`، سه Node `master1` (`192.168.72.56`)، `worker1` (`192.168.72.77`)، `worker2` (`192.168.72.106`) |
| Container runtime و CNI | containerd `2.3.7` و Cilium |
| cert-manager | Helm Release `cert-manager` در Namespace `cert-manager`؛ نسخه مورد استفاده در پروژه `v1.21.3` (با `helm list` تأیید شود) |
| Ingress Controller | **F5 NGINX Ingress Controller** Image `repo-sec.nexus.local:31920/nginx/nginx-ingress:5.7.0`؛ Chart مشاهده‌شده `nginx-ingress-2.8.0` |
| IngressClass | `nginx-f5`، Controller ID: `nginx.org/ingress-controller` |
| Ingress Service | `nginx-ingress-controller`، `NodePort`: HTTP `30080` و HTTPS `30443` |
| Ingress scheduling | یک Replica روی `master1` و `externalTrafficPolicy: Local`؛ برای تست فعلی IP مستر مناسب است |
| Issuer | `ClusterIssuer/cluster-ca-issuer` در وضعیت `Ready=True`؛ `ClusterIssuer/root-selfsigned` در وضعیت `Ready=True` |
| Root CA | `Certificate/cluster-root-ca` و Secret `cluster-root-ca-secret` در Namespace `cert-manager` |
| TLS سایت | `Certificate/cert-security`، `Secret/cert-security-tls` در `nginx-ingress`، `Ready=True` |
| Ingress | `nginx-ingress/cert-security-ingress` با Host `cert.security.local` |
| Backend آزمایشی | Service و Deployment به نام `security-demo` در Namespace `nginx-ingress`، Port `80` |

**نکته:** گواهی قدیمی `jenkins-cert` در Namespace `test-cert` صرفاً تست پیشین بود و برای Ingress فعلی استفاده نشده است. یک CA آزمایشی جداگانه با نام `demo-internal-ca` نیز در کلاستر موجود بود، ولی مسیر واقعی صدور `cert-security` بر اساس پیکربندی پیشنهادی، `cluster-ca-issuer` است. با دستورهای بخش 9 زنجیره واقعی را تأیید کنید.

### جریان درخواست

```text
Client / Browser
    | https://cert.security.local:30443
    | TLS؛ بررسی Host/SAN و اعتماد به Root CA
    v
192.168.72.56:30443 (NodePort, externalTrafficPolicy=Local)
    v
F5 NGINX Ingress Controller (nginx-f5)
    | spec.tls.secretName: cert-security-tls
    | TLS termination
    v
Service/security-demo:80
    | HTTP (نه HTTPS داخلی)
    v
Pod/security-demo
```

cert-manager **به درخواست هر مرورگر پاسخ نمی‌دهد**؛ گواهی را صادر/تمدید و در Kubernetes Secret قرار می‌دهد. NGINX هنگام اتصال مرورگر گواهی را ارائه می‌کند. اتصال فعلی بین Ingress و backend از نوع HTTP است؛ برای رمزنگاری داخلی باید HTTPS upstream / mTLS را جداگانه طراحی کرد.

## 2. پیش‌نیازها و احتیاطات

- `kubectl` با دسترسی مدیریتی و Helm 3 روی ماشین مدیریت؛ Registry داخلی `repo-sec.nexus.local:31920` با Imageهای لازم.
- Helm Chartهای دانلودشده و Mirror Imageها باید **هم‌نسخه** باشند؛ روی کلاستر Offline هیچ Pull اینترنتی انجام نشود.
- **Private Key** مربوط به Root CA را در Git، CI artifacts یا فایل‌های اشتراکی قرار ندهید. Secretهای Kubernetes به‌شکل Base64 رمزنگاری نشده‌اند.
- وضعیت PKI فعلی را پیش از Apply هر نمونه بررسی کنید؛ ساخت مجدد Root CA ممکن است اعتماد سرویس‌های موجود را بشکند.
- `*.local` می‌تواند با mDNS تعارض داشته باشد. برای محیط عملیاتی از زیردامنه مدیریت‌شده سازمان استفاده شود.
- یک CA داخلی فقط وقتی در مرورگر Trusted است که **گواهی عمومی CA** در Trust Store کلاینت نصب شده باشد.
- فعلاً NodePort HTTPS `30443` است؛ بدون NAT/LB/VIP یا تغییر معماری، `https://cert.security.local` به‌صورت پیش‌فرض روی 443 کار نخواهد کرد.

## 3. نصب آفلاین cert-manager

در پروژه از Chart `cert-manager-v1.21.3.tgz` و Mirror Imageهای `controller`، `webhook`، `cainjector` و `acmesolver` استفاده شده بود. دستور زیر الگوی نصب است؛ **از Helm values واقعی همان Release** استفاده کنید:

```bash
kubectl create namespace cert-manager --dry-run=client -o yaml | kubectl apply -f -
helm show chart ./cert-manager-v1.21.3.tgz
helm install cert-manager ./cert-manager-v1.21.3.tgz \
  -n cert-manager --set crds.enabled=true -f cert-manager-offline-values.yaml
```

`cert-manager-offline-values.yaml` باید متناسب با مسیر Imageهای Mirrorشده، `imagePullSecrets` و تنظیمات CRD تنظیم شود؛ این فایل اصلی در اطلاعات موجود نیست و در این مخزن جعل نشده است. اگر Release قبلاً موجود است **دستور install را تکرار نکنید**.

```bash
helm list -n cert-manager
helm get values cert-manager -n cert-manager -o yaml
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager.io
```

> نام `installCRDs` در بعضی نسخه‌ها/نمونه‌های قدیمی دیده می‌شود، اما برای Chart فعلی `crds.enabled=true` گزینه مستند است. هنگام بازتولید حتماً خروجی `helm show values` همان Chart را بررسی کنید.

## 4. ایجاد Root CA و ClusterIssuer (نمونه بازتولیدی؛ روی CA فعلی دوباره اجرا نشود)

مفهوم‌ها:

- **Root CA:** گواهی با `isCA: true` که برای امضای گواهی سرویس‌ها استفاده می‌شود. Secret آن شامل **Private Key حساس** است.
- **SelfSigned Issuer:** فقط برای Bootstrap گواهی Root CA استفاده می‌شود.
- **CA ClusterIssuer:** از CA Secret امضاکننده استفاده می‌کند و برای Namespaceهای مختلف Certificate صادر می‌کند.
- **Certificate:** درخواست گواهی برای Host معین؛ cert-manager آن را در Secret نوع `kubernetes.io/tls` می‌نویسد.

### ترتیب ایجاد در یک کلاستر تازه

```bash
kubectl apply -f manifests/01-root-selfsigned.yaml
kubectl apply -f manifests/02-root-ca-certificate.yaml
kubectl -n cert-manager wait --for=condition=Ready certificate/cluster-root-ca --timeout=180s
kubectl apply -f manifests/03-cluster-ca-issuer.yaml
kubectl get clusterissuer cluster-ca-issuer
```

`ClusterIssuer` از Secret در **Cluster Resource Namespace** می‌خواند (به‌صورت پیش‌فرض `cert-manager`، مگر تغییر در تنظیمات Controller). صرفاً داشتن Secret در Namespace برنامه کافی نیست.

**برای نصب موجود:** فقط بررسی کنید:

```bash
kubectl get clusterissuer -o wide
kubectl -n cert-manager get certificate,secret
kubectl get clusterissuer cluster-ca-issuer -o yaml
```

### ملاحظات PKI

در الگوی فوق Root CA با Private Key داخل Kubernetes ایجاد می‌شود؛ برای Lab قابل استفاده است، اما برای Production طراحی PKI بهتر است Root CA آفلاین باشد و امضای روزمره با Intermediate CA یا PKI/HSM سازمان انجام شود. برای چرخه تمدید، توزیع Trust، CRL/OCSP در صورت نیاز، Backup امن کلید، محدودسازی RBAC و Rotation برنامه داشته باشید. `Ready=True` به معنای Trusted بودن گواهی در مرورگر نیست.

## 5. نصب آفلاین F5 NGINX Ingress Controller

در خروجی واقعی مشاهده شد:

```text
Image: repo-sec.nexus.local:31920/nginx/nginx-ingress:5.7.0
Chart: nginx-ingress-2.8.0
IngressClass: nginx-f5
Service: NodePort; 80:30080, 443:30443
externalTrafficPolicy: Local
Controller replicas: 1
```

برای بازتولید نمونه، Chart محلی `nginx-ingress-2.8.0.tgz` و فایل نمونه `values/nginx-ingress-values.example.yaml` را فراهم کنید. **این فایل نمونه به‌جای Helm values اصلی نیست** و گزینه‌ها باید در `helm show values` برای همین Chart تأیید شوند.

```bash
kubectl create namespace nginx-ingress --dry-run=client -o yaml | kubectl apply -f -
helm show chart ./nginx-ingress-2.8.0.tgz
helm show values ./nginx-ingress-2.8.0.tgz > chart-defaults.yaml
# فقط برای نصب تازه، پس از بازبینی و تطبیق:
helm install nginx-ingress ./nginx-ingress-2.8.0.tgz \
  -n nginx-ingress -f values/nginx-ingress-values.example.yaml
```

روی کلاستر فعلی فقط وضعیت را استخراج کنید:

```bash
helm list -n nginx-ingress
helm get values nginx-ingress -n nginx-ingress -o yaml
kubectl get ingressclass -o wide
kubectl -n nginx-ingress get deploy,pods,svc
kubectl -n nginx-ingress get svc nginx-ingress-controller -o yaml
```

در Deployment فعلی `-enable-cert-manager=false` دیده شد. این گزینه مانع استفاده از TLS Secret استاندارد `spec.tls.secretName` در یک Ingress معمولی نمی‌شود. ما Certificate را **صریحاً** ایجاد کرده‌ایم و controller صرفاً Secret را مصرف می‌کند.

## 6. صدور گواهی جداگانه برای cert.security.local

در Namespace `nginx-ingress`:

```bash
kubectl apply -f manifests/04-cert-security.yaml
kubectl -n nginx-ingress wait --for=condition=Ready certificate/cert-security --timeout=180s
kubectl -n nginx-ingress get certificate cert-security
kubectl -n nginx-ingress get secret cert-security-tls
```

در کلاستر مشاهده شده: `Certificate/cert-security` برابر `Ready=True` و Secret از نوع `kubernetes.io/tls` است.

**قاعده Namespace:** `Ingress` و Secret گواهی TLS باید در **یک Namespace** باشند. برای یک برنامه در Namespace دیگر، Certificate جداگانه با همان `ClusterIssuer` درخواست کنید؛ Secret را به صورت پیش‌فرض از Namespace دیگر نمی‌توان ارجاع داد.

## 7. Backend آزمایشی و Ingress HTTPS

Manifestهای زیر الگوی آزمایش هستند. Image `repo-sec.nexus.local:31920/library/nginx:alpine` باید از پیش در Registry داخلی موجود باشد یا با Image nginx داخلی موجود جایگزین شود:

```bash
kubectl apply -f manifests/05-security-demo.yaml
kubectl -n nginx-ingress rollout status deploy/security-demo
kubectl -n nginx-ingress get endpoints security-demo
kubectl apply -f manifests/06-cert-security-ingress.yaml
kubectl -n nginx-ingress get ingress cert-security-ingress
kubectl -n nginx-ingress describe ingress cert-security-ingress
```

فایل Ingress مشخص می‌کند:

- `ingressClassName: nginx-f5`
- Host برابر `cert.security.local`
- TLS Secret برابر `cert-security-tls`
- Backend Service برابر `security-demo:80`
- برای Redirect مستقیم HTTP به HTTPS، Annotation مناسب F5 NGINX، `nginx.org/ssl-redirect` است. **با `nginx.ingress.kubernetes.io/...` متعلق به کنترلر دیگر اشتباه نشود.**

**هشدار NodePort:** Redirect معمولاً به پورت استاندارد `443` می‌رود؛ تا وقتی انتقال پورت 443 آماده نیست، تست HTTPS را مستقیماً روی `30443` انجام دهید.

## 8. تست با Browser بدون DNS Server

روی Client ویندوزی با دسترسی Administrator فایل زیر را باز کنید:

```text
C:\Windows\System32\drivers\etc\hosts
```

خط زیر را اضافه کنید:

```text
192.168.72.56    cert.security.local
```

CMD:

```cmd
ipconfig /flushdns
```

سپس در Browser:

```text
https://cert.security.local:30443
```

به‌دلیل `externalTrafficPolicy: Local` و تک‌Replica بودن کنترلر روی `master1`، آدرس Master برای تست انتخاب شده است. با جابه‌جایی Pod یا تغییر سیاست سرویس، این فرض باید دوباره بررسی شود.

## 9. راستی‌آزمایی TLS، SAN و CA Chain

```bash
# تست TLS و مسیریابی؛ -k اعتبارسنجی گواهی را دور می‌زند
curl -vk --resolve cert.security.local:30443:192.168.72.56 \
  https://cert.security.local:30443/

# بررسی گواهی که واقعا از Ingress ارائه می‌شود
openssl s_client -connect 192.168.72.56:30443 \
  -servername cert.security.local -showcerts </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# استخراج فقط گواهی عمومی Root CA (نه tls.key)
kubectl -n cert-manager get secret cluster-root-ca-secret \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > root-ca.crt
chmod 644 root-ca.crt

# تست واقعی Trust Chain و hostname بدون -k
curl -v --cacert ./root-ca.crt \
  --resolve cert.security.local:30443:192.168.72.56 \
  https://cert.security.local:30443/
```

اگر گواهی با CA میانی صادر شده، باید Bundle مناسب شامل زنجیره درست تهیه شود. از موفقیت `curl -k` نمی‌توان نتیجه گرفت CA مورد اعتماد است. علامت Secure در Browser هم با خروجی Trust و Hostname بررسی شود.

برای مشاهده Issuer واقعی Certificate:

```bash
kubectl -n nginx-ingress get certificate cert-security \
  -o jsonpath='{.spec.issuerRef.name}{"\n"}{.spec.dnsNames}{"\n"}'
```

## 10. Security Checklist و Troubleshooting

- [ ] cert-manager (`controller`, `webhook`, `cainjector`) و Ingress Controller آماده‌اند.
- [ ] `cluster-ca-issuer` و `cert-security` در وضعیت `Ready=True` هستند.
- [ ] IngressClass دقیقاً `nginx-f5` است.
- [ ] `cert-security-tls` و Ingress در Namespace یکسان `nginx-ingress` هستند.
- [ ] SAN گواهی شامل `DNS:cert.security.local` است.
- [ ] `curl --cacert` بدون `-k` موفق است.
- [ ] Root CA عمومی روی Clientها Trusted است؛ Private Key هیچ‌جا در Git نیست.
- [ ] Redirect پورت 30080 به 443/30443 متناسب با شبکه بررسی شده است.
- [ ] Backend در این سناریو HTTP است، نه End-to-End TLS.
- [ ] پشتیبان‌گیری و Rotation CA/Issuer، ثبت تغییرات، محدودسازی RBAC و Policy لازم تعریف شده است.

دستورهای بررسی خطا:

```bash
kubectl -n nginx-ingress describe ingress cert-security-ingress
kubectl -n nginx-ingress get events --sort-by=.lastTimestamp | tail -30
kubectl -n nginx-ingress logs deploy/nginx-ingress-controller --tail=100
kubectl -n cert-manager logs deploy/cert-manager --tail=100
kubectl -n nginx-ingress describe certificate cert-security
kubectl -n nginx-ingress get endpoints security-demo
```

## 11. اسناد و تنظیمات اصلی را چگونه بدون لو دادن Secretها ثبت کنیم؟

```bash
helm get values cert-manager -n cert-manager -o yaml > cert-manager-values.review.yaml
helm get values nginx-ingress -n nginx-ingress -o yaml > nginx-ingress-values.review.yaml
kubectl get clusterissuer root-selfsigned cluster-ca-issuer -o yaml > issuers.review.yaml
kubectl -n cert-manager get certificate cluster-root-ca -o yaml > root-ca-certificate.review.yaml
kubectl -n nginx-ingress get certificate cert-security -o yaml > tls-certificate.review.yaml
kubectl -n nginx-ingress get ingress cert-security-ingress -o yaml > tls-ingress.review.yaml
```

قبل از Commit باید خروجی‌های Live را بازبینی و `managedFields`، `resourceVersion`، `uid`، Annotationهای حساس و هر Credential احتمالی را حذف کنید. **از `kubectl get secret -o yaml` برای Commit استفاده نکنید.**

## 12. منابع رسمی

- cert-manager Helm: https://cert-manager.io/docs/installation/helm/
- SelfSigned CA bootstrap: https://cert-manager.io/docs/configuration/selfsigned/
- CA Issuer و مدیریت Trust: https://cert-manager.io/docs/configuration/ca/
- Issuer/ClusterIssuer: https://cert-manager.io/docs/concepts/issuer/
- F5 NGINX annotations: https://docs.nginx.com/nginx-ingress-controller/configuration/ingress-resources/advanced-configuration-with-annotations/
- F5 NGINX Helm parameters: https://docs.nginx.com/nginx-ingress-controller/install/helm/parameters/

---

**دامنه نمونه:** `cert.security.local` برای آموزش است. برای Production نام دامنه سازمانی و PKI سازمانی و دسترسی استاندارد HTTPS پورت 443 توصیه می‌شود.
