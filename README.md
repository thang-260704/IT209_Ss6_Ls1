# Bài 1: Khảo sát FHS và Phân quyền File/Folder nâng cao

## Mục tiêu
- Tạo và điều hướng thư mục trong cấu trúc cây thư mục Linux (**FHS**: Filesystem Hierarchy Standard).
- Phân quyền bằng **số (octal)** và bằng **ký tự (symbolic)**.
- Thay đổi chủ sở hữu (owner) và nhóm sở hữu (group).

## Môi trường

| Thông tin | Giá trị |
|---|---|
| Máy chủ | VPS Ubuntu 22.04.5 LTS – `160.187.229.88` |
| User thao tác (non-root) | `devops` (có quyền `sudo`) |
| Nhóm sở hữu | `www-data` (nhóm mặc định của web server Nginx trên Ubuntu) |

---

## Vì sao đặt ứng dụng ở `/var/www`?

Theo chuẩn FHS:

| Thư mục | Vai trò |
|---|---|
| `/etc` | File cấu hình hệ thống (ví dụ `/etc/nginx`) |
| `/var` | Dữ liệu **thay đổi** trong quá trình chạy: log, cache, web content |
| `/var/www` | Nơi quy ước chứa mã nguồn website do web server phục vụ |
| `/var/log` | Log của hệ thống và dịch vụ |
| `/home` | Thư mục cá nhân của người dùng thường |

Vì vậy `/var/www/my-app` là vị trí đúng chuẩn cho ứng dụng web.

---

## Các bước thực hiện

Đăng nhập vào máy chủ bằng user thường:

```bat
ssh -i %USERPROFILE%\.ssh\id_ed25519 -p 2222 devops@160.187.229.88
```

### Bước 1 – Tạo cấu trúc thư mục

```bash
sudo mkdir -p /var/www/my-app/public
sudo mkdir -p /var/www/my-app/logs
```

`-p` tạo luôn các thư mục cha nếu chưa có, và không báo lỗi nếu thư mục đã tồn tại.

### Bước 2 – Đổi chủ sở hữu và nhóm sở hữu

```bash
sudo chown -R devops:www-data /var/www/my-app
```

- `devops` là **owner**, một tài khoản non-root.
- `www-data` là **group**, để tiến trình Nginx (chạy bằng user `www-data`) đọc được mã nguồn và ghi được log.
- `-R` áp dụng **đệ quy** cho toàn bộ thư mục con.

### Bước 3 – Phân quyền thư mục `public`, dùng ký hiệu **symbolic**

```bash
sudo chmod u=rwx,g=rx,o= /var/www/my-app/public
```

Lệnh này tương đương với `chmod 750`.

### Bước 4 – Phân quyền thư mục `logs`, dùng **số octal**

```bash
sudo chmod 770 /var/www/my-app/logs
```

Lệnh này tương đương với `chmod u=rwx,g=rwx,o=`.

### Bước 5 – Kiểm tra

```bash
ls -la /var/www/my-app
id devops
```

---

## Kết quả

```text
devops@wean2607:~$ ls -la /var/www/my-app
total 16
drwxr-xr-x 4 devops www-data 4096 Oct  6 21:51 .
drwxr-xr-x 7 root   root     4096 Oct  6 21:50 ..
drwxrwx--- 2 devops www-data 4096 Oct  6 21:51 logs
drwxr-x--- 2 devops www-data 4096 Oct  6 21:50 public
```

![ls -la /var/www/my-app](images/01-ls-la.png)

| Thư mục | Quyền hiển thị | Octal | Owner | Group |
|---|---|---|---|---|
| `public` | `drwxr-x---` | `750` | `devops` | `www-data` |
| `logs` | `drwxrwx---` | `770` | `devops` | `www-data` |

---

## Giải thích cơ chế phân quyền

Chuỗi `drwxr-x---` được đọc như sau:

```text
 d   rwx   r-x   ---
 │    │     │     └── Others: không có quyền
 │    │     └──────── Group  : đọc + vào thư mục
 │    └────────────── Owner  : đọc + ghi + vào thư mục
 └─────────────────── d = directory
```

| Ký hiệu | Giá trị | Với **file** | Với **thư mục** |
|---|---|---|---|
| `r` | 4 | Đọc nội dung | Liệt kê danh sách file (`ls`) |
| `w` | 2 | Sửa nội dung | Tạo, xoá, đổi tên file bên trong |
| `x` | 1 | Thực thi | **Đi vào thư mục** (`cd`) và truy cập file bên trong |

Cách tính octal là cộng các giá trị: `rwx = 4+2+1 = 7`, `r-x = 4+1 = 5`, `--- = 0`. Từ đó suy ra **750** và **770**.

> **Lưu ý về đề bài:** phần ràng buộc ghi "Owner đọc và ghi, Group chỉ đọc" cho thư mục `public`, nhưng kết quả mong đợi là `drwxr-x---` (750). Với **thư mục**, quyền `x` là bắt buộc để có thể truy cập vào bên trong. Nếu chỉ đặt `640` (`rw-r-----`), cả owner lẫn group đều **không `cd` vào được** và không đọc được file trong đó. Vì vậy bài làm dùng **750** theo đúng kết quả mong đợi.

### Octal và Symbolic

| Cách viết | Ví dụ | Đặc điểm |
|---|---|---|
| Octal | `chmod 770 logs` | Đặt **toàn bộ** quyền một lần, ngắn gọn |
| Symbolic, gán tuyệt đối | `chmod u=rwx,g=rx,o= public` | Dễ đọc, rõ từng đối tượng |
| Symbolic, thêm hoặc bớt | `chmod g+w file`, `chmod o-r file` | Chỉ thay đổi đúng một quyền, giữ nguyên các quyền khác |

### Vì sao thiết kế như vậy?

- **`public` (750)**: chỉ owner `devops` được sửa mã nguồn. Nginx (`www-data`) chỉ cần đọc để phục vụ, nên Nginx không thể bị lợi dụng để sửa code. Others bị chặn hoàn toàn.
- **`logs` (770)**: ứng dụng hoặc web server thuộc nhóm `www-data` cần **ghi** log, nên group có toàn quyền. Others không đọc được log, tránh lộ thông tin nhạy cảm.

## Đối chiếu yêu cầu

| Yêu cầu | Kết quả |
|---|---|
| Tạo `/var/www/my-app/public` và `/var/www/my-app/logs` | ✅ |
| `public` = `drwxr-x---` (750) | ✅ |
| `logs` = `drwxrwx---` (770) | ✅ |
| Owner là user non-root (`devops`), group là `www-data` | ✅ |
| Dùng cả phân quyền octal và symbolic | ✅ |
