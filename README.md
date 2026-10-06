# 🚀 Docker Swarm & CI/CD Zero-Downtime Deployment Lab

Bu proje, modern DevOps süreçlerini, yüksek erişilebilirlik (High Availability) prensiplerini ve CI/CD otomasyonunu pratik etmek amacıyla Rocky Linux üzerinde kurguladığım 3 node'lu bir Docker Swarm laboratuvar ortamıdır.

Amacım; kodun GitHub deposuna gönderilmesinden (push) canlı sunucularda kesintisiz (zero-downtime) ayağa kalkmasına kadar olan tüm süreci insan müdahalesi olmadan otomatize etmek ve ortamı metrik, log ve alarm katmanlarıyla izlenebilir hale getirmektir.

**Öne çıkanlar**

- Her `push` ile otomatik build, güvenlik taraması ve deploy
- `start-first` rolling update ve sağlık kontrolü ile kesintisiz güncelleme, hatalı sürümde otomatik rollback
- Her imaj commit SHA ile etiketlenir, hangi sürümün canlıda olduğu izlenebilir
- Prometheus + Grafana (metrik), Loki + Alloy (log), Alertmanager (e-posta alarmı)
- Portainer ile merkezi yönetim

## 🛠️ Kullanılan Teknolojiler (Tech Stack)

| Katman | Teknoloji |
|---|---|
| **İşletim Sistemi** | Rocky Linux (1 Manager, 2 Worker) |
| **Orkestrasyon** | Docker Swarm (stack dosyası ile) |
| **CI/CD** | GitHub Actions (Self-hosted runner) |
| **Konteyner Registry** | GitHub Container Registry (GHCR) |
| **İmaj Güvenlik Taraması** | Trivy |
| **Metrik İzleme** | Prometheus, Node Exporter, Grafana |
| **Loglama** | Loki, Grafana Alloy |
| **Alarm** | Alertmanager (e-posta bildirimi) |
| **Gizli Bilgi Yönetimi** | Docker Secrets |
| **Yönetim Arayüzü** | Portainer |

---

## 🏗️ 1. Altyapı ve Cluster Mimarisi

Sistem, 3 adet Rocky Linux sunucusundan oluşmaktadır. Tüm düğümler (nodes) aktif olarak trafiği karşılamaktadır.

```mermaid
flowchart LR
    Dev[Geliştirici] -->|git push main| GH[GitHub]
    GH -->|tetikler| Runner[Self-hosted Runner]
    Runner -->|build + Trivy taraması| Runner
    Runner -->|docker push| GHCR[(GHCR)]
    Runner -->|docker stack deploy| Mgr[Swarm Manager]
    Mgr --> W1[Worker 1]
    Mgr --> W2[Worker 2]
    GHCR -.->|imaj çekme| W1
    GHCR -.->|imaj çekme| W2

    subgraph Gözlemlenebilirlik
        Prom[Prometheus] --> Graf[Grafana]
        Alloy[Alloy - her node] --> Loki[Loki]
        Loki --> Graf
        Prom -->|alarm kuralları| AM[Alertmanager]
        AM -->|e-posta| Mail[Mail Sunucusu]
    end
```

<img width="821" height="92" alt="Docker Swarm node listesi" src="https://github.com/user-attachments/assets/8e6547ab-a662-40b4-9dd5-8fef11e483be" />

---

## 🔄 2. CI/CD Pipeline ve Kesintisiz Dağıtım

Yazılımın canlı ortama alınması tamamen GitHub Actions üzerinden kurgulanmıştır.
Geliştirici kodu `main` dalına (branch) gönderdiğinde pipeline otomatik olarak tetiklenir:

1. Repo çekilir ve Docker imajı **commit SHA** ve `latest` etiketleriyle derlenir.
2. İmaj **Trivy** ile taranır, kritik ve düzeltilebilir bir açık varsa pipeline durur.
3. İmaj **GHCR**'a (GitHub Container Registry) yüklenir.
4. Self-hosted runner, Swarm cluster'ına `docker stack deploy` ile yeni sürümü uygular (Rolling Update).
5. Smoke test uygulanır. Servis ayağa kalkmazsa **otomatik rollback** yapılır.

**Kesintisiz güncelleme için kullanılan ayarlar**

- `order: start-first`: Yeni container sağlıklı hale gelmeden eskisi kapatılmaz.
- `healthcheck`: Swarm trafiği yalnızca sağlıklı container'lara yönlendirir.
- `parallelism: 1`: Replica'lar tek tek güncellenir.
- `failure_action: rollback`: Başarısız güncelleme otomatik geri alınır.

**Yapılandırma dosyaları:** [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) (pipeline) ve [`stack.yml`](stack.yml) (Swarm servis tanımı).

### Başarılı Pipeline Geçmişi

<img width="1166" height="559" alt="Başarılı GitHub Actions pipeline geçmişi" src="https://github.com/user-attachments/assets/7be94947-f65e-4b07-b078-a1b35e46ceef" />

---

## 📊 3. Monitoring, Loglama ve Alarm

Sistemi üç katmanda izliyorum: **metrik** (ne oluyor), **log** (neden oluyor) ve **alarm** (biri bakmadan haberdar olma).

Aşağıdaki CLI çıktısında görüldüğü üzere; web uygulaması 2 replica ile yük dengelemesi (load balancing) yaparak çalışırken, Node Exporter metrik toplamak için "global" modda her sunucuya (3/3) otomatik olarak dağıtılmıştır.

<img width="1322" height="198" alt="docker service ls çıktısı" src="https://github.com/user-attachments/assets/2ab798cc-848b-4b0c-99a4-da314916de7b" />

### Portainer ile Merkezi Yönetim

Servislerin sağlığını, loglarını ve replica durumlarını anlık izlemek için Portainer kullanılmaktadır.

<img width="1640" height="510" alt="Portainer dashboard" src="https://github.com/user-attachments/assets/d53cbed8-473f-40cb-a985-ab3bcedd54b9" />

### Grafana & Prometheus ile Metrik İzleme

CPU, RAM, Network ve Disk kullanımları Node Exporter ile toplanır, Prometheus'ta saklanır ve Grafana dashboard'unda görselleştirilir.

<img width="1913" height="795" alt="Grafana dashboard" src="https://github.com/user-attachments/assets/ee24f4c9-41c9-4650-982e-f6314fbb803b" />

### Loki & Alloy ile Merkezi Loglama

Container logları her node'da çalışan **Grafana Alloy** ajanı tarafından toplanıp **Loki**'ye gönderilir. Bir container hangi node'da çalışırsa çalışsın, logları Grafana'daki **Explore** ekranından tek yerden incelenebilir.

<img width="1917" height="910" alt="image" src="https://github.com/user-attachments/assets/69f472ca-080e-4641-9946-3b0a831a600c" />


> Not: Promtail, Grafana tarafından 2 Mart 2026'da kullanımdan kaldırıldığı için log ajanı olarak Alloy tercih edilmiştir.

### Alertmanager ile Alarm

Prometheus'taki alarm kuralları tetiklendiğinde **Alertmanager** bildirimi e-posta ile gönderir. SMTP kimlik bilgisi yapılandırma dosyasında düz metin olarak durmaz, **Docker secret** olarak tanımlanmıştır.

<img width="443" height="365" alt="Alarm bildirimi ekran görüntüsü" src="https://github.com/user-attachments/assets/ebfdd580-0232-4cef-a610-5a7b78b7ba90" />

---

## 🔒 4. Ağ ve Güvenlik

- **En az yetki ilkesi:** Workflow izinleri yalnızca `contents: read` ve `packages: write` ile sınırlıdır.
- **Gizli bilgi yönetimi:** Registry girişi için geçici `GITHUB_TOKEN` kullanılır, repoda sabit şifre yoktur. Mail sunucusu şifresi Docker secret olarak saklanır.
- **İmaj taraması:** Her build Trivy ile taranır.
- **Kaynak sınırları:** Servisler için CPU ve bellek limitleri tanımlıdır.
- **Firewall:** Sunucularda `firewalld` aktiftir. Swarm için gereken portlar açıktır: `2377/tcp` (cluster yönetimi), `7946/tcp+udp` (node'lar arası iletişim) ve `4789/udp` (overlay ağ trafiği).

---

## 🧭 5. Bilinen Sınırlamalar

Bu bir laboratuvar ortamıdır ve bilinçli olarak kabul edilen sınırlamaları vardır:

- **Tek manager node:** Manager düşerse çalışan servisler devam eder, ancak cluster yönetilemez ve yeni deploy yapılamaz.
- **Tek self-hosted runner:** Runner düşerse pipeline çalışmaz.

---

*Bu laboratuvar ortamı, modern DevOps süreçlerini pratik etmek amacıyla hazırlanmıştır.*
