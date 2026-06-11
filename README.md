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

## B. THỰC HÀNH

> *(Cập nhật sau)*
