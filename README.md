# Graylog, MongoDB & OpenSearch Centralized Logging Sistemi

## 📘 Overview
Bu proje; sistemlerden gelen ham ve dağınık logları merkezi, düzenli ve anlamlı hale getirmek amacıyla kurulmuştur. Graylog ve OpenSearch kullanarak loglar tek bir yerde toplanmış; pipeline ve stream'ler ile işlenip görselleştirilebilir hale getirilir.

Bu altyapı sayesinde loglar sadece okunur metinler olmaktan çıkıp filtrelenebilir, analiz edilebilir ve izlenebilir hale gelerek sağlam bir **Observability (Gözlemlenebilirlik)** temeli oluşturur.

---

## ⚙️ Architecture & Workflow
Sistem, yüksek erişilebilirlik ve performans için şu işleyiş sırasına göre çalışmaktadır:

1. **Ingestion:** Client/Server/Agent log gönderir → Graylog Input.  
2. **Processing:** Graylog logu alır → Journal'a (buffer) yazar → Pipeline/Stream ile işler.  
3. **Storage:** İşlenen log verisi → OpenSearch'e indexlenir.  
4. **Visualization:** Kullanıcı UI'dan arama yapar → Graylog, OpenSearch'e sorgu atar ve sonuçlar gösterilir.  
5. **Metadata:** Kullanıcı, stream, pipeline ve alert ayarları → MongoDB'de saklanır.

---

## 🧩 Components
| Component | Purpose |
| :--- | :--- |
| **OpenSearch** | Logları kalıcı saklayan, indeksleyen ve hızlı arama sağlayan motor. |
| **Graylog** | Log toplama, işleme, zenginleştirme ve görselleştirme arayüzü. |
| **MongoDB** | Graylog'un yapılandırma ve metadata bilgilerini saklayan veritabanı. |
| **Pipelines** | Ham logları anlamlı alanlara ayırmak ve etiketlemek için kullanılan motor. |

---

## 🚀 Deployment Guide

### 1. Pre-requisites (Tüm Node'lar)
Kuruluma başlamadan önce tüm sunucularda aşağıdaki hazırlıkları tamamlayın.

- OpenSearch sunucularında /etc/hosts ayarı:
```text
192.168.100.X  opensearch-1
192.168.100.X  opensearch-2
192.168.100.X  opensearch-3
```

- Docker kurulumu:
Docker kurulumu tüm sunucularda gereklidir. Başlamadan önce güncellemelerin yapılması ve docker'ın yüklenmesi önerilir.

```bash
sudo apt update && sudo apt -y install ca-certificates curl gnupg
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER && newgrp docker
```

- Firewall (Private Subnet) Ayarı:
TCP 9200 — OpenSearch API  
TCP 9300 — Node-to-node cluster communication  
TCP 22   — SSH

- Sistem optimizasyonu (OpenSearch için):
```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### 2. OpenSearch Cluster Kurulumu (3 Node)
Her sunucuda `node.name` ve IP adreslerini kendi yapınıza göre güncelleyerek çalıştırın.

```bash
docker volume create osdata
docker run -d --name opensearch --restart unless-stopped -p 9200:9200 -p 9300:9300 \
  -e "cluster.name=demo-os" -e "node.name=os1" \
  -e "network.host=0.0.0.0" \
  -e "network.publish_host=192.168.100.x" \
  -e "transport.publish_host=192.168.100.x" \
  -e "discovery.seed_hosts=192.168.100.x,192.168.100.x,192.168.100.x" \
  -e "cluster.initial_master_nodes=os1,os2,os3" \
  -e "OPENSEARCH_JAVA_OPTS=-Xms256m -Xmx256m" \
  -e "DISABLE_SECURITY_PLUGIN=true" \
  --ulimit nofile=65536:65536 -v osdata:/usr/share/opensearch/data \
  opensearchproject/opensearch:2.19.4
```

- Sağlık kontrolleri (OpenSearch için):
```bash
docker ps
curl http://localhost:9200
curl "http://localhost:9200/_cluster/health?pretty"
curl "http://localhost:9200/_cat/nodes?v"
```

- Disk watermark ayarları (OpenSearch için):
```bash
curl -X PUT "http://localhost:9200/_cluster/settings" \
  -H 'Content-Type: application/json' \
  -d '{
    "persistent": {
      "cluster.routing.allocation.disk.watermark.low": "90%",
      "cluster.routing.allocation.disk.watermark.high": "95%",
      "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
    }
  }'
```

### 3. MongoDB Cluster Kurulumu (3 Node)
MongoDB, Graylog’un kullanıcılar, stream’ler, pipeline kuralları ve dashboard ayarları gibi metadata bilgilerini saklamak için kullanıldı.

```bash
docker volume create mongo_data

docker run -d --name mongo \
  --restart unless-stopped \
  --network host \
  -v mongo_data:/data/db \
  mongo:6 \
  mongod --replSet rs0 --bind_ip_all --port 27017
```

- Replica set kontrolü:
Replica setini kontrol etmemizin sebebi cluster yapımızı onaylamaktır. Output olarak 1 PRIMARY 2 SECONDARY beklenir.

```bash
docker exec -it mongo mongosh
docker exec -it mongo mongosh --eval \
  "rs.status().members.map(m=>({name:m.name,state:m.stateStr}))"
```

### 4. Graylog Kurulumu
Her sunucuda `node.name` ve IP adreslerini kendi yapınıza göre güncelleyerek çalıştırın.

Kısaca hizmet portları ve kullanım:
- 9000 → Graylog Web UI
- 12201/udp → GELF UDP input
- 1514/udp → syslog UDP (kullanırsanız)

Graylog, MongoDB URI ve OpenSearch endpoint'lerini kullanır (Graylog eski isimlendirmede Elasticsearch host'ları bekler).

```bash
docker run -d --name graylog \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 12201:12201/udp \
  -p 1514:1514/udp \
  -e GRAYLOG_PASSWORD_SECRET="1a96d046306e199ffb0b7a4284caae4b95707747084a1492562a15535c90544941905" \
  -e GRAYLOG_ROOT_PASSWORD_SHA2="3eb3fe66b31e3b4d10fa70b5cad49c7112294af6ae4e476a1c405155d45aa12131" \
  -e GRAYLOG_HTTP_BIND_ADDRESS="0.0.0.0:9000" \
  -e GRAYLOG_MONGODB_URI="mongodb://192.168.100.x:27017,192.168.100.x:27017,192.168.100.x:27017/graylog?replicaSet=rs0" \
  -e GRAYLOG_ELASTICSEARCH_HOSTS="http://192.168.100.x:9200,http://192.168.100.x:9200,http://192.168.100.x:9200" \
  graylog/graylog:6.1
```

- Graylog ayarlamaları ve gönderim testi:
Graylog’un düzgün bir şekilde çalışabilmesi için input konfigürasyonu yapılması gerekir.

UI → System → Inputs
- Select input: GELF UDP
- Node: (kendi Graylog node’un)
- Port: 12201
- Title: gelf-udp
- Start

Test log gönderimi:
```bash
echo -n '{"version":"1.1","host":"graylog","short_message":"pipeline test","level":3}' \
  | nc -u -w1 127.0.0.1 12201

echo -n '{"version":"1.1","host":"graylog","short_message":"ilk log geldi","level":3}' \
  | nc -u -w1 127.0.0.1 12201
```

- Graylog UI üzerinden kontrol:
UI → Search  
Gönderilen mesaj metni aratılır. Mesaj görünüyorsa log ingestion başarılıdır.

- Opsiyonel pipeline ve stream ayarları:
Amaç: Graylog’un merkezi işleme (processing) yeteneklerini göstermek.

Akış: Pipeline → Stream → Index

PART A — Stream oluşturma  
UI → Streams → Create stream

Örnek:
- Name: API-Errors
- Rule: level >= 3 veya message contains "ERROR"

PART B — Pipeline Rule oluşturma  
UI → System → Pipelines → Create Rule

Kural örneği:
- _service field’ı varsa service field’ına kopyala
- Log’a tag ekle

Edit connections bölümünden oluşturulan stream ile pipeline arasındaki bağlantıyı kurun.

---

## 🧑‍💻 Author
Created by **Umut Can** — DevOps Automation & Cloud Infrastructure Project  
© 2025 — All rights reserved.
