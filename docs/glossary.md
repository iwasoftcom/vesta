# vesta docs — translation glossary

Consistent terminology across all `documentation.<lang>.html` files. Code,
field names, API operation names (`PutObject`, `SigV4`…), and product names
stay in English in every language. Where a term is conventionally kept in
English in technical writing (token, endpoint, webhook…), keep it.

Source is English (`docs/i18n/documentation.en.html`). Each translation is a
copy of that file with prose translated in place; CSS, code blocks, and SVG
stay byte-identical.

| English | tr | de | fr | es | zh | ar |
|---|---|---|---|---|---|---|
| object store | nesne deposu | Objektspeicher | stockage d'objets | almacenamiento de objetos | 对象存储 | تخزين الكائنات |
| bucket | bucket | Bucket | bucket | bucket | 存储桶 | دلو (bucket) |
| object | nesne | Objekt | objet | objeto | 对象 | كائن |
| control plane | kontrol düzlemi | Kontrollebene | plan de contrôle | plano de control | 控制平面 | مستوى التحكم |
| data plane | veri düzlemi | Datenebene | plan de données | plano de datos | 数据平面 | مستوى البيانات |
| coordinator | koordinatör | Koordinator | coordinateur | coordinador | 协调器 | منسّق |
| multipart upload | çok parçalı yükleme | Multipart-Upload | téléversement multipart | carga multiparte | 分片上传 | تحميل متعدد الأجزاء |
| versioning | versiyonlama | Versionierung | versionnage | versionado | 版本控制 | إصدارات |
| delete marker | silme işaretçisi | Löschmarkierung | marqueur de suppression | marcador de eliminación | 删除标记 | علامة حذف |
| Object Lock / WORM | nesne kilidi / WORM | Objektsperre / WORM | verrouillage d'objet / WORM | bloqueo de objeto / WORM | 对象锁定 / WORM | قفل الكائن / WORM |
| legal hold | yasal tutma | Rechtliche Aufbewahrung | conservation légale | retención legal | 法律保留 | حجز قانوني |
| erasure coding | silme kodlama | Erasure Coding | codage à effacement | codificación de borrado | 纠删码 | ترميز المحو |
| encryption at rest | bekleme durumunda şifreleme | Verschlüsselung im Ruhezustand | chiffrement au repos | cifrado en reposo | 静态加密 | تشفير أثناء الخزن |
| rate limiting | hız sınırlama | Ratenbegrenzung | limitation de débit | limitación de tasa | 速率限制 | تحديد المعدل |
| token bucket | token kovası | Token-Bucket | seau à jetons | cubo de tokens | 令牌桶 | دلو الرموز |
| consensus | konsensüs | Konsens | consensus | consenso | 共识 | إجماع |
| leader election | lider seçimi | Leader-Wahl | élection du leader | elección de líder | 领导选举 | انتخاب القائد |
| replication | replikasyon | Replikation | réplication | replicación | 复制 | تكرار |
| quorum | çoğunluk (quorum) | Quorum | quorum | quórum | 法定人数 | نصاب |
| admin console | yönetim konsolu | Admin-Konsole | console d'administration | consola de administración | 管理控制台 | لوحة الإدارة |
| tenant | tenant (kiracı) | Mandant | tenant (locataire) | inquilino (tenant) | 租户 | مستأجر |
| quota | kota | Kontingent | quota | cuota | 配额 | حصة |
| bucket policy | bucket policy'si | Bucket-Richtlinie | politique de bucket | política de bucket | 存储桶策略 | سياسة الدلو |
| lifecycle rule | yaşam döngüsü kuralı | Lifecycle-Regel | règle de cycle de vie | regla de ciclo de vida | 生命周期规则 | قاعدة دورة الحياة |
| inventory report | envanter raporu | Bestandsbericht | rapport d'inventaire | informe de inventario | 清单报告 | تقرير الجرد |
| event bus | olay veriyolu | Event-Bus | bus d'événements | bus de eventos | 事件总线 | ناقل الأحداث |
| webhook | webhook | Webhook | webhook | webhook | Webhook | ويب هوك |
| metadata search | metadata araması | Metadaten-Suche | recherche de métadonnées | búsqueda de metadatos | 元数据搜索 | بحث البيانات الوصفية |
| transform-on-read | okuma-anında dönüştürme | Transform-on-Read | transformation à la lecture | transformación en lectura | 读时转换 | تحويل عند القراءة |
| downloads | indirmeler | Downloads | téléchargements | descargas | 下载 | التنزيلات |
| source code | kaynak kod | Quellcode | code source | código fuente | 源代码 | الشيفرة المصدرية |

Notes:
- Product/proper names unchanged: **Vesta, S3, SigV4, AWS, Raft, openraft,
  Reed–Solomon, AES-256-GCM, MCP, Docker, K8s, TLS, gRPC, JSON, XML**.
- Arabic (`ar`) files set `<html lang="ar" dir="rtl">`; `<pre>`/`<code>`
  blocks keep `dir="ltr"`.
- Chinese (`zh`) is LTR.
