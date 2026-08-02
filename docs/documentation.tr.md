Vesta

eksikleri kapatan S3-uyumlu nesne deposu · v0.1.0

Rust ile yazılmış, katmanlı, S3-uyumlu bir object storage sistemi — tek bir ikili dosya, yazılımı değiştirmeden dizüstü bilgisayardan Raft-replikeli bir cluster'a kadar ölçeklenir.

**AI ile mi geliştiriyorsun?** Kodlama ajanına/LLM'ine bu sayfa yerine şu linki ver — kurulum, her ortam değişkeni ve tam varsayılanı, birebir API çağrılarıyla yoğun/makine-optimize bir referans; ajan bunu tersine mühendislik yapmak zorunda kalmadan doğrudan kullanır: [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## Vesta nedir

Vesta, günümüzün nesne depolama çözümlerinde (S3, GCS, Azure Blob, R2, B2, Wasabi, MinIO, Ceph, SeaweedFS, Garage) görülen özellik eksiklerini hedef alır. Gerçek S3 API'sini konuşur — SigV4 imzalama, çok parçalı yükleme, versiyonlama, koşullu istekler, toplu silme — ve **kontrol düzlemini** (metadata: bucket'lar, nesne indeksi, IAM) **veri düzleminden** (diskteki içerik-adresli bloklar) tamamen ayırır, böylece her biri bağımsız olarak değiştirilebilir veya ölçeklenebilir.

## Tasarım ilkeleri

**Kontrol düzlemi / veri düzlemi ayrımı.**  
Metadata ve byte'lar ayrı trait sınırlarının arkasında yaşar. Depolama motorları, konsensüs arka uçları ve şifreleme katmanları S3 API katmanına dokunmadan değiştirilir.

**Config dosyasında unutulmuş bir yönetim düğmesi kalmaz.**  
Hız sınırları, GC aralıkları, CORS, kotalar ve politika çalışma-zamanı ayarlarıdır; admin konsolundan canlı değiştirilir ve replike edilir — yeniden başlatma gerektiren ortam değişkenleri değil.

**Uyumluluk bir yaklaşıklık değil, sözleşmedir.**  
SigV4 (header, presigned URL, streaming chunk), çok parçalı yükleme, versiyonlama ve koşullu istekler gerçek AWS SDK test paketlerine karşı sürekli doğrulanır — elle seçilmiş örneklere karşı değil.

## Tipik tek-ikili nesne deposundan farkı

|  | Tipik MinIO-tarzı depo | Vesta |
| --- | --- | --- |
| Konsensüs | Sabit erasure-set / gateway modeli | Dinamik üyelikli ağ-Raft — kanıtlanmış bir motor ([openraft](#architecture)) aynı yazma yolunun arkasında takılabilir, opt-in bir arka uçtur |
| Çalışma-zamanı yapılandırma | Ortam değişkenleri, değiştirmek için restart | Admin konsolu canlı ayarları (hız sınırı, GC aralığı, CORS, kota) replike log üzerinden değiştirir — restart yok |
| Metadata kalıcılığı | Arka uca göre değişir | Snapshot sıkıştırmalı append-only WAL; her düğüm bağımsız kalıcı yazar ve normal log replikasyonuyla yakalanır |
| Çok-kiracılılık | Sonradan eklenmiş veya yok | Birinci sınıf tenant'lar, tenant başına bucket kotası ve SigV4 kimlik izolasyonu |
| AI ajan erişimi | Uygulanamaz | Salt-okunur bir [MCP sunucusu](#more) native aramayı ve S3 Select'i ajan aracı olarak sunar, anahtar başına tenant izolasyonuyla |

## Hızlı başlangıç

Sunucuyu çalıştır (konteyner imajı veya bağımsız ikili):

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# veya ikili dosya
VESTA_DATA_DIR=/var/lib/vesta vesta
```

Herhangi bir S3 istemcisiyle konuş:

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://fotograflar
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://fotograflar/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://fotograflar
```

## İçindekiler

**Hız sınırlama**  
Tenant başına token-kovası, admin konsolundan canlı açılıp ayarlanır; kötü davranan istemciler bağlantı kesilmesi yerine düzgün bir `SlowDown` alır.

**Dağıtık konsensüs**  
Lider seçimi, dinamik üyelik ve kalıcı log replikasyonu olan bir ağ-Raft — veya aynı yazma yolunun arkasında kanıtlanmış bir implementasyon olan `openraft`'a opt-in ol.

**Erasure coding ve şifreleme**  
Reed–Solomon erasure-coded depolama ve AES-256-GCM bekleme-durumu şifrelemesi, ikisi de dedup-güvenli (içerik-adresli bloklar).

**Versiyonlama ve Object Lock**  
Tam versiyon geçmişi, silme işaretçileri ve yasal tutma ile WORM saklama (GOVERNANCE/COMPLIANCE).

**Çok-kiracılılık**  
Tenant'lar birinci sınıftır: tenant başına bucket kotası, SigV4 kimlik izolasyonu, bucket policy ve hazır ACL'ler.

**Arama, Select ve Lambda**  
Native ters-indeksli metadata araması, S3 Select (CSV SQL) ve okuma-anında dönüştürme (Object Lambda tarzı).

**Replikasyon ve olaylar**  
Asenkron coğrafi replikasyon, bir değişim-akışı olay veriyolu ve takılabilir webhook bildirim teslimi.

**Yaşam döngüsü ve envanter**  
Süre sonu ve depolama-sınıfı geçiş kuralları, ayrıca talep üzerine CSV envanter raporları.

## Yönetim konsolu

Ayrı, durumsuz bir yönetim uygulaması (gömülü React + MUI arayüzü) yazmaları bir depolama düğümüne proxy'ler — kendi verisini tutmaz; her değişiklik S3 API'sinin kullandığı aynı konsensüs log'u üzerinden replike edilir.

<table><tbody><tr><th>Adres</th><td><code>http://localhost:9500</code> (env <code>VESTA_ADMIN_ADDR</code>, varsayılan <code>0.0.0.0:9500</code>)</td></tr><tr><th>Bağlandığı</th><td>bir depolama düğümünün yönetim API'si, varsayılan <code>http://127.0.0.1:9000</code> (env <code>VESTA_ADMIN_NODES</code>)</td></tr><tr><th>Varsayılan kullanıcı</th><td>yok — <b>ilk</b> konsol kullanıcısı oluşturulana kadar (Kullanıcılar ekranı) konsol açık ve admin gibi davranır, ilk kullanıcı bu pencereyi kapatır. Ya da düğüm açılışında tohumla: <code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

Her düğüm aynı işlemleri düz bir JSON API olarak da sunar: `http://<düğüm>:9000/_vesta/admin/*` (konsolun kendisinin çağırdığı uçlar) — script'lemek için elverişli. İlk yapacağın üç şey:

```
# 1) Bucket oluştur
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"fotograflar"}'

# 2) Tenant oluştur (API anahtarından önce gerekli)
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) O tenant için bir API anahtarı oluştur (SigV4 access/secret ikilisi)
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

Bir konsol kullanıcısı veya API anahtarı oluşunca bu çağrılar `x-vesta-user`/`x-vesta-pass` header'ları ister (bir konsol kullanıcısının kimliği) — ve ilk anahtarın oluşması S3 API'de SigV4 zorunluluğunu küme genelinde otomatik açar, restart gerekmez.

-   **Kullanıcılar, anahtarlar ve tenant'lar** — konsol hesapları, SigV4 API anahtarları, tenant başına kotalar.
-   **Bucket'lar ve policy** — oluştur/sil, bucket policy JSON'u, public-read anahtarları.
-   **Cluster** — düğüm sağlığı, üye ekle/çıkar, minority-write ve auto-shrink anahtarları.
-   **Çalışma-zamanı ayarları** — hız sınırı, blok-GC aralığı, CORS origin: canlı değiştirilir, her düğüme replike edilir, restart'lar arasında kalıcıdır.

## Mimari

Tek bir ikili, iki ağ kapısı ve katı bir katmanlama kuralı: S3 API katmanı depolamaya asla doğrudan dokunmaz — her şey koordinatörden geçer ve küme-genelinde olması gereken her mutasyon, bir okumaya görünür olmadan önce konsensüs log'undan geçer.

S3 SDK'ları · aws-cli SigV4 · çok parçalı · versiyonlama Yönetim konsolu · AI ajanları (MCP) durumsuz proxy · tenant kapsamlı araçlar S3 API · :9000 Admin API · :9500 koordinatör (Rust): bucket · nesne · çok parçalı · policy · yaşam döngüsü · arama konsensüs log'u (özel Raft veya openraft) — mutasyonlar okumadan önce commit edilir metadata (WAL) · blok depolama (erasure-coded, şifreli, dedup'lı)

## İndirmeler ve kaynak

-   **İndirmeler:** derlenmiş artefaktlar (Windows, Debian `.deb`, RedHat `.rpm`) ve Docker imajı her versiyon için [iwasoft.com](https://iwasoft.com) → Ürünler → Vesta'da yayınlanır. Kaynak kod indirmelerin bir parçası değildir.
-   **Uyumluluk:** S3 API yüzeyi (SigV4, çok parçalı, versiyonlama, koşullu istekler) gerçek AWS SDK entegrasyon testlerine karşı sürekli doğrulanır.
-   **Dürüst durum:** erken geliştirme aşamasında — henüz bağımsız güvenlik denetiminden geçmedi, üretim kilometresi yok. Bunlar yol haritasına dair çekinceler değil, ifşalardır.

Vesta v0.1.0 · S3-uyumlu · Rust, içerik-adresli depolama, ağ-Raft (özel veya openraft). © iwasoft.
