## Mục tiêu

Thiết lập quyền cho thư mục `/var/www/ptit-web` để:

- User `devops` là chủ sở hữu và có thể đọc, sửa mã nguồn mà không cần `sudo`.
- Group `www-data` có thể đọc nội dung và truy cập các thư mục để Nginx phục vụ website tĩnh.
- Người dùng khác không có quyền truy cập.

## 1. Kiểm tra user, group và thư mục

```bash
id devops
getent group www-data
sudo test -d /var/www/ptit-web && echo "Thu muc ton tai"
```

Nếu user `devops` chưa tồn tại, tạo user trước khi tiếp tục:

```bash
sudo adduser devops
```

## 2. Thay đổi owner và group

Đổi chủ sở hữu của thư mục cùng toàn bộ nội dung bên trong thành `devops:www-data`:

```bash
sudo chown -R devops:www-data /var/www/ptit-web
```

## 3. Thiết lập quyền truy cập

Thư mục cần quyền thực thi (`x`) để user và Nginx có thể đi xuyên qua cây thư mục. File tĩnh không cần quyền thực thi, vì vậy đặt quyền riêng cho thư mục và file:

```bash
sudo find /var/www/ptit-web -type d -exec chmod 750 {} \;
sudo find /var/www/ptit-web -type f -exec chmod 640 {} \;
```

Quyền sau khi thiết lập:

| Đối tượng | Mode | User `devops` | Group `www-data` | User khác |
|---|---:|---|---|---|
| Thư mục | `750` | `rwx` | `r-x` | `---` |
| File | `640` | `rw-` | `r--` | `---` |

Trong trường hợp đề bài yêu cầu người dùng khác cũng được đọc nội dung, có thể dùng `755` cho thư mục và `644` cho file. Bài này chọn phương án an toàn hơn là không cấp quyền cho người dùng khác.

## 4. Kiểm tra kết quả phân quyền

```bash
ls -ld /var/www/ptit-web
ls -la /var/www/ptit-web/
ls -la /var/www/ptit-web/html/
```

Kết quả minh họa:

```text
drwxr-x--- 3 devops www-data 4096 Oct  1 13:30 /var/www/ptit-web

total 12
drwxr-x--- 3 devops www-data 4096 Oct  1 13:30 .
drwxr-xr-x 3 root   root     4096 Oct  1 13:00 ..
drwxr-x--- 2 devops www-data 4096 Oct  1 13:30 html

total 12
drwxr-x--- 2 devops www-data 4096 Oct  1 13:30 .
drwxr-x--- 3 devops www-data 4096 Oct  1 13:30 ..
-rw-r----- 1 devops www-data   32 Oct  1 13:30 index.html
```

> Số byte, thời gian và danh sách file có thể khác tùy máy. Các cột cần kiểm tra là quyền, owner `devops` và group `www-data`.

## 5. Kiểm tra ghi file bằng user `devops`

Nếu đang đăng nhập bằng tài khoản khác, chuyển sang user `devops`:

```bash
su - devops
whoami
echo "Update" >> /var/www/ptit-web/html/index.html
tail -n 3 /var/www/ptit-web/html/index.html
exit
```

Kết quả mong đợi:

```text
devops
Update
```

Lệnh `echo` không sử dụng `sudo` và không xuất hiện lỗi `Permission denied`, chứng tỏ user `devops` có quyền ghi file.

## 6. Kiểm tra quyền đọc của Nginx

Kiểm tra trực tiếp bằng user chạy Nginx:

```bash
sudo -u www-data test -r /var/www/ptit-web/html/index.html \
  && echo "www-data doc duoc index.html" \
  || echo "www-data khong doc duoc index.html"
```

Kết quả mong đợi:

```text
www-data doc duoc index.html
```

Kiểm tra cấu hình và tải lại Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -i http://localhost/
```

Kết quả mong đợi gồm mã trạng thái `HTTP/1.1 200 OK` và nội dung mới có dòng `Update`.

Nếu vẫn gặp lỗi `403 Forbidden`, kiểm tra Nginx đang trỏ đúng tới thư mục chứa file:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/ptit-web/html;
    index index.html;
}
```

Sau khi sửa cấu hình, chạy lại `sudo nginx -t` rồi `sudo systemctl reload nginx`.