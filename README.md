# BT5 — App Monitor & Alert Realtime

**Môn:** Phát triển ứng dụng với mã nguồn mở (TEE0421)
**Sinh viên:** Tạ Phạm Đình Hòa — MSSV: K225480106088 — Lớp: 58KTPM

## A. LÝ THUYẾT

### 1. Docker là gì?

Docker là nền tảng mã nguồn mở cho phép đóng gói ứng dụng cùng toàn bộ dependencies vào một đơn vị gọi là **container**. Container chạy độc lập, nhất quán trên mọi môi trường (laptop, server, cloud) mà không lo lỗi môi trường. Khác với máy ảo, container dùng chung kernel của host OS nên nhẹ hơn và khởi động nhanh hơn.

### 2. Các keyword trong `docker-compose.yml`

| Keyword | Ý nghĩa | Ví dụ |
|---------|---------|-------|
| `services` | Danh sách các container sẽ chạy | `services:` |
| `networks` | Mạng nội bộ giữa các container | `networks: mynet:` |
| `volumes` | Vùng lưu trữ dữ liệu dùng chung | `volumes: dbdata:` |
| `image` | Image Docker dùng để tạo container | `image: nginx:latest` |
| `container_name` | Đặt tên cố định cho container | `container_name: my-nginx` |
| `build` | Build image từ Dockerfile | `build: ./myapp` |
| `restart` | Chính sách khởi động lại | `restart: unless-stopped` |
| `ports` | Map cổng host:container | `ports: - "8080:80"` |
| `volumes` | Gắn thư mục từ host vào container | `volumes: - ./web:/usr/share/nginx/html:ro` |
| `environment` | Biến môi trường truyền vào container | `environment: - MYSQL_ROOT_PASSWORD=123` |
| `depends_on` | Service nào phải khởi động trước | `depends_on: - mariadb` |
| `networks` | Gắn service vào network | `networks: - mynet` |

**Ví dụ minh hoạ:**

```yaml
services:
  webserver:
    image: nginx:latest
    container_name: my-nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./web:/usr/share/nginx/html:ro
    depends_on:
      - mydb
    networks:
      - appnet

  mydb:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE=mydb
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - appnet

networks:
  appnet:

volumes:
  dbdata:
```

### 3. Ưu điểm khi triển khai ứng dụng bằng Docker

| # | Ưu điểm | Giải thích |
|---|---------|-----------|
| 1 | Nhất quán môi trường | Dev, test, production dùng cùng 1 image |
| 2 | Triển khai nhanh | `docker compose up -d` là xong |
| 3 | Cô lập dịch vụ | Lỗi 1 service không ảnh hưởng service khác |
| 4 | Dễ scale | Tăng số container chỉ bằng 1 lệnh |
| 5 | Dễ backup | Volume chứa data, image chứa app |
| 6 | Version control | `docker-compose.yml` lưu git được |

### 4. Triển khai lên máy chủ không có Internet

**Bước 1 — Xuất image ra file nén (trên máy có Internet)**

```bash
docker save nginx:latest mariadb:10.11 nodered/node-red:latest \
  | gzip > all_images.tar.gz
```

**Bước 2 — Copy sang máy chủ**

```bash
scp all_images.tar.gz user@192.168.1.100:/home/user/
scp -r ./myproject    user@192.168.1.100:/home/user/
```

**Bước 3 — Load image trên máy chủ**

```bash
docker load -i all_images.tar.gz
docker images
```

**Bước 4 — Chạy ứng dụng**

```bash
cd /home/user/myproject
docker compose up -d
```

> Docker dùng image đã load sẵn, không cần Internet.

## B. THỰC HÀNH — APP MONITOR CHỨNG KHOÁN VN REALTIME

### Kiến trúc hệ thống

Hệ thống gồm 7 service chạy bằng Docker Compose:

```text
Internet
    |
Cloudflare Tunnel
    |
Nginx (web server + reverse proxy)
    |-- /api/     --> Flask API --> MariaDB (giá tức thời)
    |-- /grafana/ --> Grafana   --> InfluxDB (lịch sử)
    |
Node-RED (lấy data SSI mỗi 30s)
    |-- Lưu --> MariaDB
    |-- Lưu --> InfluxDB
    |-- Alert --> Telegram Bot
```

### Bước 1 — Chuẩn bị server Ubuntu

SSH vào Ubuntu Server VM (Hyper-V):

```bash
ssh dinhhoa@192.168.1.35
lsb_release -a && docker --version && df -h /
```

![SSH vào server](images/01_ssh_server.png)

Kết quả: Ubuntu 24.04.4 LTS, Docker 29.4.0, còn 7GB dung lượng.

### Bước 2 — Tạo cấu trúc thư mục project

```bash
mkdir -p ~/bt5/{nodered,flask,nginx/myweb,mariadb,influxdb,grafana}
cd ~/bt5 && ls -la
```

![Cấu trúc thư mục](images/02_folder_structure.png)

### Bước 3 — Tạo Flask API (`flask/app.py`)

API đọc dữ liệu tức thời từ MariaDB và trả về JSON:

```python
from flask import Flask, jsonify
import pymysql, os

app = Flask(__name__)

@app.route('/api/latest')
def get_latest():
    conn = pymysql.connect(
        host=os.environ.get('DB_HOST', 'mariadb'),
        user='btuser', password='btpass', database='stockdb',
        cursorclass=pymysql.cursors.DictCursor
    )
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM stock_prices ORDER BY created_at DESC LIMIT 10")
        rows = cursor.fetchall()
    return jsonify({"status": "ok", "data": rows})
```

![Flask app.py](images/03_flask_app.png)

### Bước 4 — Tạo `docker-compose.yml`

File khai báo 7 service: mariadb, influxdb, nodered, flask, grafana, nginx, cloudflared.

```yaml
services:
  mariadb:
    image: mariadb:10.11
    container_name: bt5-mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=stockdb
      - MYSQL_USER=btuser
      - MYSQL_PASSWORD=btpass
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - bt5net

  influxdb:
    image: influxdb:2.7
    container_name: bt5-influxdb
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=adminpass
      - DOCKER_INFLUXDB_INIT_ORG=bt5org
      - DOCKER_INFLUXDB_INIT_BUCKET=stockbucket
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=bt5-super-secret-token
    networks:
      - bt5net

  nodered:
    image: nodered/node-red:latest
    container_name: bt5-nodered
    user: "1000:1000"
    volumes:
      - ./nodered:/data
    networks:
      - bt5net

  flask:
    build: ./flask
    container_name: bt5-flask
    environment:
      - DB_HOST=mariadb
      - DB_USER=btuser
      - DB_PASS=btpass
      - DB_NAME=stockdb
    networks:
      - bt5net

  grafana:
    image: grafana/grafana:latest
    container_name: bt5-grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_SECURITY_ALLOW_EMBEDDING=true
    networks:
      - bt5net

  nginx:
    image: nginx:latest
    container_name: bt5-nginx
    volumes:
      - ./nginx/myweb:/usr/share/nginx/html:ro
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - bt5net

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: bt5-cloudflared
    command: tunnel --no-autoupdate run --token <TOKEN>
    networks:
      - bt5net

networks:
  bt5net:

volumes:
  mariadb_data:
  influxdb_data:
  grafana_data:
```

### Bước 5 — Khởi động toàn bộ hệ thống

```bash
cd ~/bt5
docker compose up -d
docker compose ps
```

![7 container chạy thành công](images/05_docker_compose_up.png)

Kết quả: 7/7 container Started.

### Bước 6 — Cấu hình Cloudflare Tunnel

Tạo tunnel `bt5` trên Cloudflare dashboard, thêm 3 route:

| Subdomain | Service |
|-----------|---------|
| `bt5.taphamdinhhoa.io.vn` | `http://bt5-nginx:80` |
| `bt5-nr.taphamdinhhoa.io.vn` | `http://bt5-nodered:1880` |
| `bt5-grafana.taphamdinhhoa.io.vn` | `http://bt5-grafana:3000` |

![Cloudflare Tunnel routes](images/06_cloudflare_routes.png)

### Bước 7 — Tạo bảng MariaDB

```bash
docker exec -it bt5-mariadb mariadb -u btuser -pbtpass stockdb -e "
CREATE TABLE IF NOT EXISTS stock_prices (
  id INT AUTO_INCREMENT PRIMARY KEY,
  symbol VARCHAR(20) NOT NULL,
  price DECIMAL(15,2) NOT NULL,
  change_val DECIMAL(10,2) DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);"
```

![Tạo bảng MariaDB](images/07_create_table.png)

### Bước 8 — Cấu hình Node-RED

Truy cập `https://bt5-nr.taphamdinhhoa.io.vn`, cài 3 node:
- `node-red-node-mysql`
- `node-red-contrib-influxdb`
- `node-red-contrib-telegrambot`

Import flow gồm 7 node:
Every 30s --> SSI API --> Parse & Check Alert --> Save MariaDB --> Debug
--> Save InfluxDB
--> Check Alert --> Telegram Alert
![Node-RED flow](images/08_nodered_flow.png)

Flow hoạt động:
- Cứ 30 giây gọi API SSI lấy giá 5 mã: VCB, VIC, HPG, FPT, MBB
- Lưu vào MariaDB (giá tức thời) và InfluxDB (lịch sử)
- Nếu giá < 10.000 → ALERT LOW, giá > 100.000 → ALERT HIGH
- Gửi cảnh báo vào group Telegram

### Bước 9 — Cấu hình Grafana

Đăng nhập `http://192.168.1.51:3000` (admin/admin123), thêm datasource InfluxDB:

- Query language: Flux
- URL: `http://bt5-influxdb:8086`
- Token: `bt5-super-secret-token`
- Org: `bt5org`, Bucket: `stockbucket`

Tạo dashboard **BT5 Stock Monitor** với query:

```flux
from(bucket: "stockbucket")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "stock_prices")
  |> filter(fn: (r) => r._field == "price")
```

![Grafana dashboard](images/09_grafana_dashboard.png)

### Bước 10 — Kết quả website

Website công khai tại **https://bt5.taphamdinhhoa.io.vn**:

![Website chính](images/10_website_main.png)

Tính năng:
- Hiển thị giá cổ phiếu cập nhật mỗi 5 giây
- Badge ALERT HIGH/LOW khi vượt ngưỡng
- Iframe nhúng biểu đồ Grafana

### Bước 11 — Telegram Alert

Bot `@bt5alert_dinhhoa_bot` gửi cảnh báo vào group **Baitap5** (3 thành viên) khi phát hiện giá bất thường:

![Telegram alert](images/11_telegram_alert.png)

Nội dung alert tường minh, có giá trị gây alert:
> ⚠️ ALERT BT5
> Co phieu FPT: 138000 dong - ALERT HIGH (vuot nguong 100000)

### Bước 12 — Export và restore container

**Export toàn bộ 7 image ra file nén:**

```bash
docker save mariadb:10.11 influxdb:2.7 nodered/node-red:latest \
  bt5-flask grafana/grafana:latest nginx:latest \
  cloudflare/cloudflared:latest \
  | gzip > ~/bt5_images.tar.gz
```

File nén: `bt5_images.tar.gz` — 902MB

**Xóa toàn bộ container và image:**

```bash
docker compose down
docker system prune -a -f
```

![Xóa container](images/12_docker_prune.png)

**Load lại từ file nén:**

```bash
docker load -i ~/bt5_images.tar.gz
```

![Load lại image](images/13_docker_load.png)

**Khởi động lại:**

```bash
docker compose up -d
docker ps -a
```

![Khởi động lại thành công](images/14_docker_restore.png)

Kết quả: 7/7 container Started — hệ thống khôi phục hoàn toàn từ file nén.
