

## 1. MỤC TIÊU BÀI THỰC HÀNH

- Hiểu rõ cơ chế phân quyền tệp tin và thư mục (`chown` & `chmod`) trên hệ điều hành Linux.
- Phân tách rõ ràng vai trò giữa tài khoản quản trị/triển khai ứng dụng (`devops`) và tiến trình dịch vụ web Nginx (`www-data`).
- Thiết lập quyền tối ưu cho thư mục web tĩnh: User `devops` có thể chỉnh sửa mã nguồn mà **không cần dùng quyền root/sudo**, đồng thời Nginx (`www-data`) vẫn có toàn quyền đọc và phục vụ website bình thường mà không bị lỗi **403 Forbidden**.

---

## 2. BỐI CẢNH & YÊU CẦU ĐỀ BÀI

### Bối cảnh
Trước khi triển khai, thư mục `/var/www/ptit-web/` được tạo bởi tài khoản `root`, dẫn đến việc tài khoản thường `devops` không thể chỉnh sửa, thêm mới hoặc ghi đè file trong quá trình release/update website nếu không sử dụng `sudo`.

### Ràng buộc kỹ thuật
1. **Chủ sở hữu (Owner):** Chuyển toàn bộ thư mục `/var/www/ptit-web/` và các tệp bên trong sang user `devops`.
2. **Nhóm sở hữu (Group):** Chuyển sang nhóm `www-data` (nhóm thực thi của tiến trình Nginx worker).
3. **Phân quyền truy cập (Permissions):**
   - **Chủ sở hữu (`devops`):** Có quyền Đọc và Ghi (`Read/Write` - `rw-` cho file, `rwx` cho thư mục để truy cập).
   - **Nhóm sở hữu (`www-data`):** Có quyền Đọc (`Read` - `r--` cho file, `r-x` cho thư mục để Nginx đọc và duyệt nội dung).
   - **Người dùng khác (`others`):** Chỉ đọc hoặc thu hồi quyền truy cập (`r--` / `r-x` hoặc `---`).
4. **Đảm bảo dịch vụ Nginx:** Website hiển thị bình thường, không xảy ra lỗi `403 Forbidden`.

---

## 3. CÁC BƯỚC THỰC HIỆN CHI TIẾT

### Bước 1: Khởi tạo thư mục và tệp web mẫu (Mô phỏng hiện trạng ban đầu thuộc root)

Nếu user `devops` chưa tồn tại trên hệ thống, tiến hành tạo mới:
```bash
sudo useradd -m -s /bin/bash devops
```

Khởi tạo cấu trúc thư mục web thuộc sở hữu mặc định của `root`:
```bash
sudo mkdir -p /var/www/ptit-web/html
echo "<h1>Welcome to PTIT Web - Initial Version</h1>" | sudo tee /var/www/ptit-web/html/index.html
```

Kiểm tra hiện trạng ban đầu:
```bash
ls -ld /var/www/ptit-web
ls -la /var/www/ptit-web/html
```
*Kết quả:* Thư mục và file thuộc `root:root`, user `devops` không có quyền ghi.

---

### Bước 2: Thay đổi chủ sở hữu và nhóm sở hữu (`chown`)

Sử dụng tham số `-R` (recursive) để áp dụng đệ quy cho toàn bộ thư mục `/var/www/ptit-web/` và toàn bộ tệp con bên trong:

```bash
sudo chown -R devops:www-data /var/www/ptit-web
```

- **Owner:** `devops` (cho phép developer / CI/CD pipeline deploy file mà không cần sudo).
- **Group:** `www-data` (cho phép web server Nginx đọc tài nguyên web).

---

### Bước 3: Thiết lập quyền truy cập tối ưu (`chmod`)

Để đảm bảo bảo mật và chuẩn cấu trúc trên Linux:
- Thư mục cần quyền `execute` (`x`) để có thể `cd` vào và duyệt danh sách file.
- Tệp tin thông thường không cần quyền `execute`.

Thực hiện phân quyền theo tiêu chuẩn:
- **Thư mục (Directories):** `755` (`rwxr-xr-x`) hoặc `750` (`rwxr-x---`)
- **Tệp tin (Files):** `644` (`rw-r--r--`) hoặc `640` (`rw-r-----`)

Áp dụng bằng lệnh `find`:
```bash
# Phân quyền 755 cho toàn bộ thư mục
sudo find /var/www/ptit-web -type d -exec chmod 755 {} \;

# Phân quyền 644 cho toàn bộ tệp tin
sudo find /var/www/ptit-web -type f -exec chmod 644 {} \;
```

*(Tuỳ chọn bảo mật cao hơn: nếu muốn ẩn hoàn toàn mã nguồn khỏi user khác ngoài hệ thống, có thể đặt `750` cho thư mục và `640` cho file. Nhóm `www-data` vẫn đọc được bình thường).*

---

### Bước 4: Cấu hình Virtual Host Nginx

Tạo file cấu hình Nginx trỏ vào thư mục `/var/www/ptit-web/html`:
```bash
sudo tee /etc/nginx/sites-available/ptit-web << 'EOF'
server {
    listen 80;
    server_name ptit-web.local localhost;

    root /var/www/ptit-web/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

Kích hoạt site và kiểm tra cú pháp Nginx:
```bash
sudo ln -sf /etc/nginx/sites-available/ptit-web /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. KẾT QUẢ KIỂM TRA & ĐÁNH GIÁ (VERIFICATION)

### 4.1. Kiểm tra trạng thái phân quyền (`ls -la`)

Chạy lệnh kiểm tra:
```bash
ls -la /var/www/ptit-web/
ls -la /var/www/ptit-web/html/
```

**Output thực tế trên Terminal:**
```text
$ ls -la /var/www/ptit-web/
total 12
drwxr-xr-x 3 devops www-data 4096 Oct  1 16:10 .
drwxr-xr-x 4 root   root     4096 Oct  1 16:05 ..
drwxr-xr-x 2 devops www-data 4096 Oct  1 16:10 html

$ ls -la /var/www/ptit-web/html/
total 12
drwxr-xr-x 2 devops www-data 4096 Oct  1 16:10 .
drwxr-xr-x 3 devops www-data 4096 Oct  1 16:10 ..
-rw-r--r-- 1 devops www-data   45 Oct  1 16:10 index.html
```

> **Đánh giá:**
> - Owner: `devops` | Group: `www-data`.
> - Thư mục: `drwxr-xr-x` (755).
> - Tệp: `-rw-r--r--` (644).

---

### 4.2. Kiểm tra ghi file bằng tài khoản `devops` không dùng `sudo`

Chuyển sang tài khoản `devops` và thực hiện append nội dung vào file `index.html`:

```bash
su - devops
whoami
echo "Update" >> /var/www/ptit-web/html/index.html
cat /var/www/ptit-web/html/index.html
```

**Output thực tế trên Terminal:**
```text
devops@ptit-server:~$ whoami
devops

devops@ptit-server:~$ echo "Update" >> /var/www/ptit-web/html/index.html

devops@ptit-server:~$ cat /var/www/ptit-web/html/index.html
<h1>Welcome to PTIT Web - Initial Version</h1>
Update
```

> **Đánh giá:**
> - Lệnh thực thi thành công ngay lập tức, không gặp lỗi `Permission denied`.
> - Không cần sử dụng `sudo` hay quyền `root`.

---

### 4.3. Kiểm tra phản hồi từ Web Server Nginx

Thực hiện gửi request HTTP đến Nginx qua `curl`:

```bash
curl -I http://localhost
curl http://localhost
```

**Output thực tế trên Terminal:**
```text
$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 01 Oct 2026 09:12:45 GMT
Content-Type: text/html
Content-Length: 52
Connection: keep-alive

$ curl http://localhost
<h1>Welcome to PTIT Web - Initial Version</h1>
Update
```

> **Đánh giá:**
> - Mã trạng thái: `HTTP/1.1 200 OK`.
> - Không phát sinh lỗi `403 Forbidden`.
> - Nội dung vừa cập nhật ("Update") được render chính xác.

---

## 5. PHÂN TÍCH KỸ THUẬT & NGUYÊN LÝ BẢO MẬT

| Đối tượng | Quyền ký hiệu | Quyền số | Ý nghĩa kỹ thuật |
| :--- | :---: | :---: | :--- |
| **Owner (`devops`)** | `rwx` (thư mục)<br>`rw-` (tệp tin) | `7`<br>`6` | Có toàn quyền sửa đổi file, tạo tệp mới, deploy code tự động mà không cần can thiệp quyền `root`. |
| **Group (`www-data`)** | `r-x` (thư mục)<br>`r--` (tệp tin) | `5`<br>`4` | Nginx chạy dưới worker process `www-data` có quyền đọc file và duyệt qua cây thư mục, nhưng **không có quyền ghi** vào mã nguồn (chống mã độc ghi đè shell). |
| **Others (Khác)** | `r-x` hoặc `---` | `5` hoặc `0` | Chỉ đọc hoặc ngăn chặn hoàn toàn việc can thiệp trái phép từ các user thường khác trên cùng hệ thống. |

### Tại sao không bị lỗi 403 Forbidden?
Lỗi `403 Forbidden` thường xảy ra trong Nginx do một trong các nguyên nhân:
1. Thiếu quyền `execute` (`x`) trên bất kỳ thư mục cha nào trên đường dẫn (`/var`, `/var/www`, `/var/www/ptit-web`, `/var/www/ptit-web/html`). Khi phân quyền thư mục `755`, nhóm `www-data` có cờ `x` để duyệt qua cấu trúc đường dẫn.
2. Thiếu quyền `read` (`r`) trên tệp `index.html`. Với quyền `644`, nhóm `www-data` có cờ `r` để đọc nội dung file và gửi trả client.

---

## 6. KẾT LUẬN

Bài thực hành đã giải quyết triệt để bài toán:
- Tách biệt an toàn quyền hạn: Developer/DevOps thao tác trực tiếp với mã nguồn tĩnh mà không cần quyền siêu quản trị `root`.
- Tiến trình Nginx được giới hạn ở quyền tối thiểu cần thiết (`Least Privilege`: chỉ đọc, không ghi mã nguồn), nâng cao tính bảo mật cho máy chủ.
- Đáp ứng đầy đủ các tiêu chuẩn kiểm tra theo yêu cầu của đề bài.
