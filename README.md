# 🚀 Docker Swarm & CI/CD Zero-Downtime Deployment Lab

Bu proje, modern DevOps süreçlerini, yüksek erişilebilirlik (High Availability) prensiplerini ve CI/CD otomasyonunu pratik etmek amacıyla Rocky Linux üzerinde kurguladığım 3 node'lu bir Docker Swarm laboratuvar ortamıdır.

Projeyi oluştururken temel amacım; kodun GitHub deposuna gönderilmesinden (push) canlı sunucularda kesintisiz (zero-downtime) ayağa kalkmasına kadar olan tüm süreci insan müdahalesi olmadan otomatize etmektir.

## 🛠️ Kullanılan Teknolojiler (Tech Stack)

| Katman | Teknoloji |
|---|---|
| **İşletim Sistemi** | Rocky Linux (1 Manager, 2 Worker) |
| **Orkestrasyon** | Docker Swarm |
| **CI/CD** | GitHub Actions (Self-hosted runner) |
| **Container Registry** | GitHub Container Registry (GHCR) |
| **İzleme (Monitoring)** | Prometheus, Grafana, Node Exporter |
| **Yönetim Arayüzü** | Portainer |

---

## 🏗️ 1. Altyapı ve Cluster Mimarisi (Infrastructure)

Sistem, olası bir sunucu çökmesine karşı ayakta kalabilmesi (HA) için 3 adet Rocky Linux sunucusundan oluşmaktadır. Tüm düğümler (nodes) aktif olarak trafiği karşılamaktadır.

<img width="821" height="92" alt="Docker Swarm node listesi" src="https://github.com/user-attachments/assets/8e6547ab-a662-40b4-9dd5-8fef11e483be" />

---

## 🔄 2. Sürekli Entegrasyon ve Dağıtım (CI/CD Pipeline)

Yazılımın canlı ortama alınması tamamen GitHub Actions üzerinden kurgulanmıştır.
Geliştirici kodu `main` dalına (branch) gönderdiğinde pipeline otomatik olarak tetiklenir:

1. Kod derlenir ve Docker imajı oluşturulur.
2. Oluşturulan imaj **GHCR**'a (GitHub Container Registry) yüklenir.
3. Self-hosted runner, Swarm cluster'ına bağlanarak `my-web-app` servisini kesintisiz (Rolling Update) olarak günceller.

### Başarılı Pipeline Geçmişi

<img width="1166" height="559" alt="Başarılı GitHub Actions pipeline geçmişi" src="https://github.com/user-attachments/assets/7be94947-f65e-4b07-b078-a1b35e46ceef" />

### GitHub Actions Workflow (`deploy.yml`) Yapılandırması

<details>
<summary>YAML dosyasını görmek için tıklayın</summary>

```yaml
name: Swarm CI/CD Pipeline

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: 1. Repoyu Çek
        uses: actions/checkout@v4

      - name: 2. GitHub Registry'ye Giriş (GHCR)
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: 3. İmajı Derle
        run: docker build -t ghcr.io/sarikayaatakan/swarm-cicd-demo:latest .

      - name: 4. İmajı GHCR'a Yükle
        run: docker push ghcr.io/sarikayaatakan/swarm-cicd-demo:latest

      - name: 5. Swarm Servisini Güncelle veya Başlat
        run: |
          if ! docker service inspect my-web-app > /dev/null 2>&1; then
            docker service create --name my-web-app --replicas 2 -p 8081:80 --with-registry-auth ghcr.io/sarikayaatakan/swarm-cicd-demo:latest
          else
            docker service update --image ghcr.io/sarikayaatakan/swarm-cicd-demo:latest --with-registry-auth --force my-web-app
          fi
```

</details>

---

## 📊 3. Servis Yönetimi ve Monitoring (İzleme)

Sistemdeki mikroservisleri ve altyapı metriklerini yönetmek için görsel arayüzler kurgulanmıştır.

Aşağıdaki CLI çıktısında görüldüğü üzere; web uygulaması 2 replica ile yük dengelemesi (load balancing) yaparak çalışırken, Node Exporter metrik toplamak için "global" modda her sunucuya (3/3) otomatik olarak dağıtılmıştır.


<img width="1048" height="161" alt="image" src="https://github.com/user-attachments/assets/4acb1d3f-9d92-4f56-a864-4cb8e8ce7968" />


### Portainer ile Merkezi Yönetim

CLI'a ek olarak, servislerin sağlığını, loglarını ve replica durumlarını anlık olarak izleyebilmek için Portainer konumlandırılmıştır.


<img width="1604" height="539" alt="image" src="https://github.com/user-attachments/assets/1bf46af0-1da0-498b-8d43-02da422b936c" />


### Grafana & Prometheus ile Proaktif İzleme

Swarm cluster'ının CPU, RAM, Network ve Disk kullanımlarını anlık olarak takip etmek için Prometheus tarafından toplanan metrikler, Grafana dashboard'u üzerinde görselleştirilmiştir.


<img width="1913" height="771" alt="image" src="https://github.com/user-attachments/assets/d28f70be-57b1-4c89-80cf-f917765e38c9" />


---
