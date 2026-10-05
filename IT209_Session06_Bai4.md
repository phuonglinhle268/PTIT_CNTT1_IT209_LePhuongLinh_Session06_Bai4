# Báo Cáo Quản Lý Tiến Trình Nền Với Nohup Và Tín Hiệu Kill

## 1. Phân tích cơ chế hoạt động của Nohup, Background Process và Tín hiệu Kill

### 1.1. Cơ chế hoạt động của `nohup` và toán tử `&`
- **Tín hiệu `SIGHUP` (Signal Hangup - Signal 1):** Khi một phiên làm việc Terminal hoặc kết nối SSH bị ngắt, tiến trình shell sẽ tự động gửi tín hiệu `SIGHUP` đến tất cả các tiến trình con do nó khởi tạo, dẫn đến việc các tiến trình này bị dừng lại.
- **Lệnh `nohup` (No Hang Up):** Cho phép tiến trình bắt (catch) và bỏ qua tín hiệu `SIGHUP`. Nhờ đó, tiến trình tiếp tục hoạt động độc lập ngay cả khi người dùng đăng xuất hoặc tắt Terminal.
- **Toán tử `&`:** Đưa tiến trình xuống thực thi dưới nền (Background Job), trả quyền điều khiển dòng lệnh lại cho người dùng ngay lập tức.
- **Chuyển hướng luồng `> /dev/null 2>&1`:** 
  - `> /dev/null`: Chuyển hướng luồng đầu ra tiêu chuẩn (stdout - file descriptor 1) vào thiết bị rỗng (null device).
  - `2>&1`: Hợp nhất luồng thông báo lỗi tiêu chuẩn (stderr - file descriptor 2) vào stdout, ngăn tiến trình in thông báo rác lên màn hình Terminal khi đang chạy ngầm.

### 1.2. Phân biệt tín hiệu `SIGTERM (15)` và `SIGKILL (9)`
- **`SIGTERM` (Signal Terminate - Signal 15):** Là tín hiệu yêu cầu dừng tiến trình tiêu chuẩn và an toàn (Graceful Shutdown). Khi nhận tín hiệu này, tiến trình có thể giải phóng bộ nhớ, đóng kết nối mạng, hoàn tất việc ghi tệp log và kết thúc đúng quy trình.
- **`SIGKILL` (Signal Kill - Signal 9):** Là tín hiệu cưỡng chế dừng tiến trình ngay lập tức do Linux Kernel trực tiếp xử lý mà tiến trình không thể bắt (catch) hay bỏ qua (ignore). Chỉ sử dụng `SIGKILL` khi tiến trình bị treo (unresponsive) hoặc không thể dừng bằng `SIGTERM`.

---

## 2. Mã nguồn kịch bản giám sát (`loop-monitor.sh`)

Kịch bản chạy vòng lặp vô tận, định kỳ mỗi 5 giây ghi nhận dấu thời gian hệ thống vào tệp tin `/tmp/monitor.log`:

```bash
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
```

---

## 3. Quy trình thực hiện chi tiết và Nhật ký thực thi (Terminal Logs)

### Bước 1: Tạo kịch bản `loop-monitor.sh` và cấp quyền thực thi
Tạo tệp kịch bản bằng HereDoc và gán quyền thực thi (`chmod +x`):
```bash
plinhle26@devops-vm1:~$ cat << 'EOF' > loop-monitor.sh
#!/bin/bash
while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
EOF
plinhle26@devops-vm1:~$ chmod +x loop-monitor.sh
plinhle26@devops-vm1:~$ ls -l loop-monitor.sh
-rwxrwxr-x 1 plinhle26 plinhle26 96 Oct  5 13:09 loop-monitor.sh
```

---

### Bước 2: Khởi chạy kịch bản ở chế độ nền độc lập bằng `nohup`
Thực thi script dưới nền, ghi nhận Job ID `[1]` và Process ID (PID) `6724`:
```bash
plinhle26@devops-vm1:~$ nohup ./loop-monitor.sh > /dev/null 2>&1 &
[1] 6724
```

---

### Bước 3: Tra cứu và xác minh số định danh tiến trình (PID)
Sử dụng công cụ `pgrep` để tìm kiếm mã PID của script đang chạy ngầm:
```bash
plinhle26@devops-vm1:~$ pgrep -f loop-monitor.sh
6724
```

---

### Bước 4: Kiểm tra tệp tin log được ghi liên tục
Sử dụng lệnh `tail -f` theo dõi nội dung tệp `/tmp/monitor.log` theo thời gian thực (ghi log đều đặn sau mỗi 5 giây):
```bash
plinhle26@devops-vm1:~$ tail -f /tmp/monitor.log
System time: Mon Oct  5 13:10:13 UTC 2026
System time: Mon Oct  5 13:10:18 UTC 2026
System time: Mon Oct  5 13:10:23 UTC 2026
System time: Mon Oct  5 13:10:28 UTC 2026
System time: Mon Oct  5 13:10:33 UTC 2026
System time: Mon Oct  5 13:10:38 UTC 2026
System time: Mon Oct  5 13:10:43 UTC 2026
System time: Mon Oct  5 13:10:48 UTC 2026
System time: Mon Oct  5 13:10:53 UTC 2026
^C
```

---

### Bước 5: Dừng tiến trình an toàn bằng tín hiệu `SIGTERM (15)`
Gửi tín hiệu `kill -15` đến PID `6724` và kiểm tra lại danh sách tiến trình qua `ps aux`:
```bash
plinhle26@devops-vm1:~$ kill -15 6724
plinhle26@devops-vm1:~$ ps aux | grep loop-monitor.sh
plinhle+    6784  0.0  0.2   4068  2080 pts/0    S+   13:11   0:00 grep loop-monitor.sh
[1]+  Terminated              nohup ./loop-monitor.sh > /dev/null 2>&1
```

**Kết quả xác minh:**
- Tiến trình `6724` đã nhận tín hiệu và kết thúc thành công với trạng thái `Terminated`.
- Lệnh `ps aux | grep loop-monitor.sh` chỉ còn lại chính tiến trình `grep`, xác nhận kịch bản `loop-monitor.sh` đã dừng hoạt động hoàn toàn.
