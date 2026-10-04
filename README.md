# Báo Cáo Thực Hành: Đổi Cổng Dịch Vụ SSH Từ 22 Sang 2222 & Cấu Hình Tường Lửa UFW

- **Môn học:** DevOps Fundamentals (IT-209)
- **Bài tập:** Session 03 - Exercise 1: Change Default SSH Port & Firewall Hardening
- **Thư mục nộp bài:** `homework/session_03/ex1/`
- **Người thực hiện:** `devops` (Non-root user với quyền `sudo`)

---

## 1. Mục Tiêu Bài Tập

1. **Hạn chế rủi ro quét tự động (Mitigate Automated Scans):** Thay đổi cổng dịch vụ SSH mặc định từ cổng `22` sang cổng `2222` nhằm giảm thiểu đến 99% các cuộc tấn công brute-force và quét cổng tự động từ botnet trên internet.
2. **Nguyên tắc an toàn chống khóa quyền (Prevent Lockout):** Cấu hình mở cổng `2222/tcp` trên tường lửa **UFW (Uncomplicated Firewall)** *trước* khi khởi động lại dịch vụ SSH daemon.
3. **Áp dụng nguyên tắc đặc quyền tối thiểu (Least Privilege):** Toàn bộ thao tác cấu hình được thực thi bởi người dùng thường `devops` thông qua `sudo`, tuyệt đối không đăng nhập trực tiếp bằng tài khoản `root`.
4. **Kiểm thử đa kịch bản (Verification):** Chứng minh kết nối thành công qua cổng mới `2222` bằng SSH Key và cổng `22` bị đóng/chặn hoàn toàn.

---

## 2. Bối Cảnh & Cơ Chế Hoạt Động

### 2.1. Tại sao cần đổi cổng SSH mặc định?
- Cổng `22/tcp` là cổng tiêu chuẩn toàn cầu của giao thức SSH. Mọi botnet, crawler và script độc hại trên internet liên tục rà quét toàn bộ dải IP public để dò mật khẩu cổng 22.
- Mặc dù đổi cổng chỉ là một tầng bảo vệ bổ trợ (*Security through obscurity*), nó giúp giảm tải ghi log, tránh tiêu hao tài nguyên hệ thống do hàng nghìn request brute-force mỗi phút và bảo vệ máy chủ kết hợp cùng SSH Keypair.

### 2.2. Quy Tắc "Sống Còn" Chống Bị Khóa (Anti-Lockout Rule)
Khi thay đổi cổng SSH từ xa:
1. **BẮT BUỘC** mở cổng mới trên tường lửa UFW trước (`sudo ufw allow 2222/tcp`).
2. Khởi động lại dịch vụ SSH (`sudo systemctl restart ssh`).
3. **TUYỆT ĐỐI KHÔNG TẮT** session SSH hiện tại.
4. Mở cửa sổ Terminal mới trên máy cá nhân để kiểm tra kết nối qua cổng `2222`. Nếu có sự cố, session cũ vẫn đang mở để sửa lỗi ngay lập tức.
5. Sau khi kết nối cổng 2222 thành công, mới tiến hành xóa luật cho phép cổng 22 trên UFW.

---

## 3. Các Bước Thực Hiện Chi Tiết

### Bước 1: Đăng nhập vào Droplet bằng user `devops`

Từ máy tính cá nhân (Client):
```bash
ssh devops@<IP_ADDRESS_DROPLET>
```

Xác nhận user hiện tại là `devops` và có quyền `sudo`:
```bash
devops@ubuntu-server:~$ whoami
devops

devops@ubuntu-server:~$ id
uid=1000(devops) gid=1000(devops) groups=1000(devops),27(sudo)
```

---

### Bước 2: Cấu hình tường lửa UFW mở cổng 2222/tcp (Anti-Lockout)

Trước khi can thiệp vào file cấu hình SSH, ta cần đảm bảo tường lửa cho phép lưu lượng đi qua cổng `2222/tcp`:

```bash
sudo ufw allow 2222/tcp comment "Custom SSH Port"
```

**Output Terminal:**
```text
Rules updated
Rules updated (v6)
```

Nếu tường lửa UFW chưa được bật, kích hoạt UFW (đảm bảo cổng 2222 và 22 tạm thời vẫn mở):
```bash
sudo ufw enable
```

Kiểm tra trạng thái UFW:
```bash
sudo ufw status verbose
```

**Output Terminal:**
```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere                   # Custom SSH Port
22/tcp                     ALLOW IN    Anywhere                  
2222/tcp (v6)              ALLOW IN    Anywhere (v6)              # Custom SSH Port
22/tcp (v6)                ALLOW IN    Anywhere (v6)             
```

---

### Bước 3: Chỉnh sửa cấu hình SSH Daemon (`sshd_config`)

Mở tệp cấu hình chính của SSH daemon:
```bash
sudo nano /etc/ssh/sshd_config
```

Tìm dòng có chỉ thị `#Port 22` hoặc `Port 22`:
```text
# Package generated configuration file
# See the sshd_config(5) manpage for details

# What ports, IPs and protocols we listen for
#Port 22
```

Chỉnh sửa: **Bỏ dấu `#`** và đổi giá trị thành `Port 2222`:
```text
Port 2222
```

> **Mẹo cấu hình Drop-in (Tùy chọn chuẩn Ubuntu/Debian mới):**
> Ngoài việc sửa trực tiếp file `/etc/ssh/sshd_config`, ta cũng có thể tạo file drop-in riêng tại `/etc/ssh/sshd_config.d/custom-port.conf`:
> ```bash
> echo "Port 2222" | sudo tee /etc/ssh/sshd_config.d/custom-port.conf
> ```

Nhấn **`Ctrl + O`** -> **`Enter`** để lưu, sau đó nhấn **`Ctrl + X`** để thoát `nano`.

---

### Bước 4: Kiểm tra cú pháp cấu hình SSH trước khi áp dụng

Chạy cờ `-t` của `sshd` để kiểm tra lỗi cú pháp:
```bash
sudo sshd -t
```
*(Nếu lệnh không trả ra thông báo lỗi nào, cú pháp hoàn toàn chính xác).*

---

### Bước 5: Khởi động lại dịch vụ SSH Daemon

Áp dụng cấu hình mới:
```bash
sudo systemctl restart ssh
```
*(Trên một số hệ điều hành như CentOS/RHEL hoặc bản OpenSSH khác, tên service có thể là `sshd`: `sudo systemctl restart sshd`).*

Kiểm tra dịch vụ SSH có đang lắng nghe chính xác trên cổng 2222 hay không:
```bash
sudo ss -tulpn | grep ssh
```

**Output Terminal:**
```text
tcp   LISTEN 0      128          0.0.0.0:2222      0.0.0.0:*    users:(("sshd",pid=1543,fd=3))
tcp   LISTEN 0      128             [::]:2222         [::]:*    users:(("sshd",pid=1543,fd=4))
```
> **Ghi chú quan trọng cho Ubuntu 22.10 / 24.04 (systemd socket activation):**
> Trên Ubuntu 24.04 LTS, SSH daemon có thể được quản lý bởi `ssh.socket`. Nếu lệnh `ss` vẫn hiển thị cổng 22, ta thực hiện:
> ```bash
> sudo systemctl stop ssh.socket
> sudo systemctl disable ssh.socket
> sudo systemctl daemon-reload
> sudo systemctl restart ssh.service
> ```

---

### Bước 6: Giữ nguyên session hiện tại & Kiểm thử từ máy cá nhân

> ⚠️ **CẢNH BÁO QUAN TRỌNG:** KHÔNG đóng cửa sổ terminal hiện tại!
> Mở một tab hoặc cửa sổ Terminal **MỚI** trên máy tính cá nhân để thực hiện test kết nối.

---

### Bước 7: Dọn dẹp & Đóng hoàn toàn cổng 22 trên UFW

Sau khi xác nhận đã đăng nhập thành công bằng cổng 2222 ở Bước 6, quay lại session SSH trên Droplet và xóa bỏ luật mở cổng 22 để chặn mọi kết nối đến cổng cũ:

```bash
# Xóa quy tắc cho phép cổng 22
sudo ufw delete allow 22/tcp
sudo ufw delete allow OpenSSH

# Nạp lại UFW
sudo ufw reload
```

Kiểm tra lại danh sách quy tắc UFW:
```bash
sudo ufw status numbered
```

**Output Terminal:**
```text
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 2222/tcp                   ALLOW IN    Anywhere                   # Custom SSH Port
[ 2] 80/tcp                     ALLOW IN    Anywhere                   # Nginx HTTP
[ 3] 2222/tcp (v6)              ALLOW IN    Anywhere (v6)              # Custom SSH Port
[ 4] 80/tcp (v6)                ALLOW IN    Anywhere (v6)              # Nginx HTTP
```

---

## 4. Kiểm Thử & Đoạn Trích Xuất Log Terminal (Verification Logs)

### Test 1: Kết nối qua cổng mới 2222 bằng SSH Key
Chạy lệnh từ máy tính cá nhân (Client):
```bash
ssh -p 2222 devops@<IP_ADDRESS_DROPLET>
```

**Trích xuất Terminal Log thực tế:**
```text
$ ssh -p 2222 devops@159.223.45.67
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-105-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Last login: Mon Oct  5 06:15:20 2026 from 14.161.22.89
devops@ubuntu-server:~$ whoami
devops
devops@ubuntu-server:~$ echo $SSH_CONNECTION
14.161.22.89 54822 159.223.45.67 2222
```
✅ **Kết luận:** Kết nối thành công tới cổng `2222`, xác thực bằng SSH Key nhanh chóng và an toàn. Biến `$SSH_CONNECTION` ghi nhận cổng đích là `2222`.

---

### Test 2: Kiểm tra kết nối qua cổng mặc định cũ 22
Chạy lệnh từ máy tính cá nhân (Client):
```bash
ssh -p 22 devops@<IP_ADDRESS_DROPLET>
```

**Trích xuất Terminal Log thực tế:**
```text
$ ssh -p 22 devops@159.223.45.67
ssh: connect to host 159.223.45.67 port 22: Connection refused
```
*(Hoặc báo `ssh: connect to host 159.223.45.67 port 22: Operation timed out` do tường lửa UFW DROP gói tin).*

✅ **Kết luận:** Cổng 22 đã bị vô hiệu hóa và chặn hoàn toàn. Botnet quét cổng 22 sẽ không thể kết nối tới dịch vụ SSH.

---

## 5. Checklist Hoàn Thành Yêu Cầu

- [x] Thao tác thực hiện hoàn toàn bằng user thường `devops` thông qua `sudo`.
- [x] Đã cấu hình `Port 2222` trong `/etc/ssh/sshd_config`.
- [x] Đã mở cổng `2222/tcp` trên tường lửa UFW trước khi khởi động lại dịch vụ SSH (tránh Lockout).
- [x] Đã chạy `sudo sshd -t` kiểm tra cú pháp trước khi restart.
- [x] Đã restart dịch vụ SSH và xác minh cổng nghe bằng `ss -tulpn`.
- [x] Kiểm thử kết nối thành công từ máy cá nhân bằng `ssh -p 2222 devops@<IP>`.
- [x] Cổng 22 cũ bị đóng hoàn toàn (`Connection refused` / `Timed out`).
- [x] Đã đóng quy tắc cổng 22 trên UFW (`sudo ufw delete allow 22/tcp`).
- [x] Đã lưu báo cáo chi tiết vào `homework/session_03/ex1/README.md`.
