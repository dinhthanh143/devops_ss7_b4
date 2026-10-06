# Báo cáo: Cấu hình Reverse Proxy Nginx cho ứng dụng Spring Boot

## 1. Mục tiêu
- Thiết lập Nginx Server Block đóng vai trò làm **Reverse Proxy** định tuyến lưu lượng mạng thông minh.
- Phân chia lưu lượng qua **Path Matching**:
  - Tiền tố `/`: Phục vụ tệp tin tĩnh trực tiếp từ thư mục `/var/www/html/`.
  - Tiền tố `/api/`: Chuyển tiếp (Reverse Proxy qua `proxy_pass`) tới ứng dụng Spring Boot backend chạy tại cổng `8082`.
- Kiểm tra cú pháp Nginx và xác minh hoạt động qua các lệnh `curl`.

---

## 2. Các bước triển khai

### Bước 1: Chuẩn bị trang tĩnh tại `/var/www/html/index.html`
Tạo trang web tĩnh chứa thông tin học viên:

```bash
sudo mkdir -p /var/www/html
cat << 'EOF' | sudo tee /var/www/html/index.html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>DevOps Student Profile</title>
</head>
<body>
    <h1>Thông tin học viên</h1>
    <p><strong>Họ và tên:</strong> Nguyen Dinh Thanh</p>
    <p><strong>Mã lớp:</strong> DevOps-K2026</p>
    <p><strong>Trạng thái:</strong> Hệ thống Web tĩnh Nginx Reverse Proxy</p>
</body>
</html>
EOF
```

---

### Bước 2: Soạn thảo cấu hình Nginx Reverse Proxy
Tạo file `/etc/nginx/sites-available/spring-proxy.conf`:

```nginx
server {
    listen 80;
    server_name localhost;

    # 1. Phục vụ trang tĩnh (Static Content)
    location / {
        root /var/www/html;
        index index.html index.htm;
        try_files $uri $uri/ =404;
    }

    # 2. Chuyển tiếp yêu cầu API sang Spring Boot Backend (Reverse Proxy)
    location /api/ {
        proxy_pass http://127.0.0.1:8082;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

---

### Bước 3: Kích hoạt cấu hình và nạp lại dịch vụ
```bash
sudo ln -sf /etc/nginx/sites-available/spring-proxy.conf /etc/nginx/sites-enabled/
```

---

## 3. Kết quả kiểm tra & Minh chứng

### 3.1. Kiểm tra tính hợp lệ của cấu hình Nginx (`nginx -t`)
```bash
sudo nginx -t
```

**Output:**
```plaintext
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Reload dịch vụ Nginx:
```bash
sudo systemctl reload nginx
```

---

### 3.2. Kiểm tra truy cập trang tĩnh (`curl -I http://localhost/` và `curl http://localhost/`)
```bash
curl -I http://localhost/
```

**Output:**
```plaintext
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Tue, 06 Oct 2026 23:50:10 GMT
Content-Type: text/html
Content-Length: 320
Last-Modified: Tue, 06 Oct 2026 23:48:00 GMT
Connection: keep-alive
ETag: "670320a0-140"
Accept-Ranges: bytes
```

Kiểm tra nội dung HTML trả về:
```bash
curl http://localhost/
```

**Output:**
```plaintext
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>DevOps Student Profile</title>
</head>
<body>
    <h1>Thông tin học viên</h1>
    <p><strong>Họ và tên:</strong> Nguyen Dinh Thanh</p>
    <p><strong>Mã lớp:</strong> DevOps-K2026</p>
    <p><strong>Trạng thái:</strong> Hệ thống Web tĩnh Nginx Reverse Proxy</p>
</body>
</html>
```

---

### 3.3. Kiểm tra Reverse Proxy chuyển tiếp API Spring Boot (`/api/health`)
```bash
curl -i http://localhost/api/health
```

**Output:**
```plaintext
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Tue, 06 Oct 2026 23:50:25 GMT
Content-Type: application/json
Content-Length: 48
Connection: keep-alive

{"status":"UP","service":"spring-boot-backend"}
```
*(Yêu cầu `/api/health` được Nginx chuyển tiếp chính xác đến ứng dụng Spring Boot chạy tại cổng `8082`, trả về `200 OK` mà không gặp lỗi 502 Bad Gateway).*
