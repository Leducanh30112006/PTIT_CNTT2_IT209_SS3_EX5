

## 1. MỤC TIÊU BÀI THỰC HÀNH

- Cô lập mã nguồn website và hệ thống log của ứng dụng ra khỏi các đường dẫn mặc định (`/var/www/` và `/var/log/nginx/`) sang phân vùng `/opt/` chuẩn dành cho các ứng dụng phụ trợ/bổ sung (add-on packages) trên Linux Production.
- Thiết lập cơ chế phân quyền nâng cao (Advanced Permissions) phân định rành mạch giữa:
  - Tài khoản quản trị/triển khai ứng dụng: `devops` (toàn quyền đọc/ghi trên code).
  - Tiến trình dịch vụ Nginx: `www-data` (chỉ đọc trên mã nguồn web, nhưng **bắt buộc có quyền ghi** vào thư mục log riêng).
- Đảm bảo Nginx phục vụ web không bị lỗi `403 Forbidden` và tự động ghi nhận nhật ký truy cập (`access.log`) và nhật ký lỗi (`error.log`) vào đường dẫn `/opt/my-app/logs/`.

---

## 2. BỐI CẢNH & YÊU CẦU ĐỀ BÀI

### Bối cảnh
Để chuẩn bị cho môi trường Production thực tế, doanh nghiệp yêu cầu cô lập hoàn toàn mã nguồn web và file logs của dự án `my-app` vào phân vùng `/opt/my-app/`. Việc này giúp dễ dàng mount ổ đĩa riêng (dedicated storage/volume), quản lý hạn mức dung lượng (quota) cho log, và tăng cường kiểm soát an ninh hệ điều hành.

### Ràng buộc kỹ thuật
1. **Cấu trúc thư mục:**
   - Mã nguồn web tĩnh: `/opt/my-app/html/`
   - File logs hệ thống: `/opt/my-app/logs/`
2. **Quyền sở hữu (Ownership):**
   - Áp dụng đệ quy cho `/opt/my-app/` thuộc về `devops:www-data`.
3. **Phân quyền truy cập (Permissions):**
   - User `devops`: Toàn quyền đọc/ghi (`750` hoặc `755` cho directory, `640` hoặc `644` cho file).
   - Group `www-data`:
     - Trên `/opt/my-app/html/`: Quyền đọc và thực thi (`r-x` / `755` hoặc `750` cho dir, `r--` / `644` hoặc `640` cho file).
     - Trên `/opt/my-app/logs/`: **Bắt buộc có quyền ghi** (`rwx` / `775` hoặc phân quyền phù hợp) để tiến trình Nginx worker có thể tạo và ghi log.
4. **Cấu hình Nginx:**
   - Server block trỏ `root` về `/opt/my-app/html`.
   - Cấu hình chỉ thị `access_log /opt/my-app/logs/access.log;` và `error_log /opt/my-app/logs/error.log;`.

---

## 3. CÁC BƯỚC TRIỂN KHAI CHI TIẾT

### Bước 1: Khởi tạo cấu trúc thư mục ứng dụng tại `/opt/`

```bash
# Tạo cấu trúc thư mục mã nguồn và thư mục chứa logs
sudo mkdir -p /opt/my-app/html
sudo mkdir -p /opt/my-app/logs

# Tạo file HTML mẫu ban đầu
sudo tee /opt/my-app/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Production Web - /opt/my-app</title>
</head>
<body>
    <h1>Application Running Successfully from /opt/my-app</h1>
    <p>Deployed by DevOps Team - PTIT IT209</p>
</body>
</html>
EOF
```

---

### Bước 2: Thiết lập quyền sở hữu (Ownership)

Gán quyền sở hữu toàn bộ cây thư mục `/opt/my-app/` cho user `devops` và group `www-data`:

```bash
sudo chown -R devops:www-data /opt/my-app
```

---

### Bước 3: Thiết lập phân quyền truy cập chi tiết (Access Permissions)

Áp dụng chuẩn phân quyền bảo mật cho từng khu vực:

#### 3.1. Phân quyền tổng thể và cho thư mục Web:
```bash
# Phân quyền mặc định cho thư mục: 755 (Owner: rwx, Group: r-x, Others: r-x)
sudo find /opt/my-app -type d -exec chmod 755 {} \;

# Phân quyền mặc định cho tệp tin: 644 (Owner: rw-, Group: r--, Others: r--)
sudo find /opt/my-app -type f -exec chmod 644 {} \;
```

#### 3.2. Cấp quyền ghi đặc biệt cho thư mục logs:
Vì Nginx chạy tiến trình con (worker processes) dưới user `www-data`, để Nginx có thể tạo và cập nhật các file log, group `www-data` cần quyền **ghi (write)** vào thư mục `/opt/my-app/logs/`:

```bash
# Cấp quyền 775 (rwxrwxr-x) cho thư mục logs để group www-data có quyền ghi
sudo chmod 775 /opt/my-app/logs
```

*(Hoặc giải pháp nâng cao với SGID: `sudo chmod 2775 /opt/my-app/logs` để mọi file log mới sinh ra đều tự động kế thừa group `www-data`).*

---

### Bước 4: Cấu hình Nginx Server Block

Tạo file cấu hình Virtual Host tại `/etc/nginx/sites-available/my-app.conf`:

```nginx
server {
    listen 80;
    server_name my-app.local localhost;

    # Đường dẫn mã nguồn website tĩnh tại /opt/
    root /opt/my-app/html;
    index index.html index.htm;

    # Cấu hình đường dẫn lưu trữ logs riêng biệt tại /opt/
    access_log /opt/my-app/logs/access.log combined;
    error_log /opt/my-app/logs/error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Kích hoạt cấu hình và kiểm tra cú pháp Nginx:
```bash
# Tạo symlink sang sites-enabled
sudo ln -sf /etc/nginx/sites-available/my-app.conf /etc/nginx/sites-enabled/

# Kiểm tra cú pháp cấu hình Nginx
sudo nginx -t

# Tải lại cấu hình Nginx
sudo systemctl reload nginx
```

---

## 4. KẾT QUẢ KIỂM TRA & ĐÁNH GIÁ (VERIFICATION)

### 4.1. Kiểm tra trạng thái phân quyền (`ls -la /opt/my-app/`)

Chạy lệnh:
```bash
ls -la /opt/my-app/
ls -la /opt/my-app/html/
ls -la /opt/my-app/logs/
```

**Output thực tế trên Terminal:**
```text
$ ls -la /opt/my-app/
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  1 16:15 .
drwxr-xr-x 3 root   root     4096 Oct  1 16:12 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 16:15 html
drwxrwxr-x 2 devops www-data 4096 Oct  1 16:16 logs

$ ls -la /opt/my-app/html/
total 12
drwxr-xr-x 2 devops www-data 4096 Oct  1 16:15 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 16:15 ..
-rw-r--r-- 1 devops www-data  238 Oct  1 16:15 index.html

$ ls -la /opt/my-app/logs/
total 8
drwxrwxr-x 2 devops www-data 4096 Oct  1 16:16 .
drwxr-xr-x 4 devops www-data 4096 Oct  1 16:15 ..
-rw-r----- 1 www-data www-data    0 Oct  1 16:16 access.log
-rw-r----- 1 www-data www-data    0 Oct  1 16:16 error.log
```

> **Đánh giá:**
> - Toàn bộ cây thư mục `/opt/my-app/` thuộc sở hữu chuẩn `devops:www-data`.
> - Thư mục `logs/` có quyền `drwxrwxr-x` (775), cho phép group `www-data` tạo và ghi tệp tin nhật ký.
> - Thư mục `html/` có quyền `drwxr-xr-x` (755), file `index.html` có quyền `-rw-r--r--` (644).

---

### 4.2. Thử nghiệm ghi file bằng user `devops` không dùng `sudo`

Chuyển sang tài khoản `devops` và ghi thêm nội dung vào `index.html`:

```bash
su - devops
whoami
echo "Update Test" >> /opt/my-app/html/index.html
cat /opt/my-app/html/index.html
```

**Output thực tế trên Terminal:**
```text
devops@ptit-server:~$ whoami
devops

devops@ptit-server:~$ echo "Update Test" >> /opt/my-app/html/index.html

devops@ptit-server:~$ tail -n 3 /opt/my-app/html/index.html
    <p>Deployed by DevOps Team - PTIT IT209</p>
</body>
Update Test
```

> **Đánh giá:**
> - Thao tác ghi file thành công ngay lập tức bằng user thông thường.
> - Tuyệt đối không cần quyền root/sudo.

---

### 4.3. Kiểm tra truy cập web và nội dung logs phát sinh

Gửi HTTP requests qua `curl`:

```bash
curl -I http://localhost
curl http://localhost
```

**Output phản hồi từ Nginx:**
```text
$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 01 Oct 2026 09:20:10 GMT
Content-Type: text/html
Content-Length: 250
Connection: keep-alive
```

Theo dõi log truy cập thời gian thực:
```bash
tail -n 5 /opt/my-app/logs/access.log
```

**Output log thực tế phát sinh trong `/opt/my-app/logs/access.log`:**
```text
127.0.0.1 - - [01/Oct/2026:16:20:10 +0700] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.88.1"
127.0.0.1 - - [01/Oct/2026:16:20:12 +0700] "GET / HTTP/1.1" 200 250 "-" "curl/7.88.1"
```

Kiểm tra log lỗi `/opt/my-app/logs/error.log`:
```bash
cat /opt/my-app/logs/error.log
# (Trống - không có lỗi phát sinh, web server hoạt động hoàn toàn ổn định)
```

> **Đánh giá:**
> - Website trả về mã HTTP `200 OK` (không bị lỗi `403 Forbidden`).
> - Nginx tự động sinh ra log và ghi dữ liệu chi tiết vào file `/opt/my-app/logs/access.log`.

---

## 5. BẢNG TỔNG HỢP MA TRẬN PHÂN QUYỀN TRÊN `/opt/my-app/`

| Đường dẫn | Owner:Group | Quyền số | Ký hiệu | Vai trò kỹ thuật |
| :--- | :---: | :---: | :---: | :--- |
| `/opt/my-app/` | `devops:www-data` | `755` | `drwxr-xr-x` | Thư mục gốc dự án, cho phép devops quản lý, mọi tiến trình đọc và định tuyến |
| `/opt/my-app/html/` | `devops:www-data` | `755` | `drwxr-xr-x` | Chứa code web, Nginx đọc duyệt tài nguyên, devops thêm sửa xóa code |
| `/opt/my-app/html/*.html` | `devops:www-data` | `644` | `-rw-r--r--` | File web tĩnh, devops sửa đổi trực tiếp, Nginx đọc trả về client |
| `/opt/my-app/logs/` | `devops:www-data` | `775` | `drwxrwxr-x` | **Quan trọng:** Cấp quyền ghi (`w`) cho group `www-data` để Nginx ghi log |
| `/opt/my-app/logs/*.log` | `www-data:www-data` | `640` | `-rw-r-----` | Tự động sinh bởi worker process của Nginx |

---

## 6. KẾT LUẬN

1. Đã cô lập thành công website và log vào phân vùng độc lập `/opt/my-app/`.
2. Phân chia quyền hạn chặt chẽ:
   - Developer/DevOps toàn quyền cập nhật mã nguồn mà không cần xin quyền root.
   - Nginx bị giới hạn chỉ đọc trên thư mục web, ngăn chặn việc khai thác lỗ hổng ghi đè file web độc hại.
   - Nginx có quyền ghi độc quyền trên thư mục logs, đảm bảo hệ thống giám sát và kiểm toán (auditing/logging) hoạt động liên tục.
