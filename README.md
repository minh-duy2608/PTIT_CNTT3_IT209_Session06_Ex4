# Bài 4 - Quản lý tiến trình nền với `nohup` và tín hiệu `kill`

## 1. Mục tiêu

Chạy script giám sát ở chế độ nền, giữ cho script tiếp tục hoạt động sau khi đóng phiên SSH, tìm PID và dừng tiến trình an toàn bằng tín hiệu `SIGTERM`.

## 2. Nội dung script

File [`loop-monitor.sh`](./loop-monitor.sh) ghi thời gian hệ thống vào `/tmp/monitor.log` sau mỗi 5 giây:

```bash
#!/bin/bash

trap 'exit 0' TERM INT

while true; do
    echo "System time: $(date)" >> /tmp/monitor.log
    sleep 5
done
```

Lệnh `trap` giúp script kết thúc có kiểm soát khi nhận `SIGTERM` hoặc `SIGINT`.

## 3. Cấp quyền và chạy nền bằng `nohup`

Từ thư mục `homework/session_06/ex4/`, thực hiện:

```bash
chmod +x loop-monitor.sh
nohup ./loop-monitor.sh > /dev/null 2>&1 &
MONITOR_PID=$!
echo "PID: $MONITOR_PID"
```

Trong đó:

- `nohup` làm cho tiến trình bỏ qua tín hiệu `SIGHUP` khi phiên Terminal/SSH đóng.
- `> /dev/null 2>&1` chuyển hướng cả đầu ra chuẩn và lỗi chuẩn.
- `&` đưa tiến trình xuống chạy nền.
- `$!` là PID của tiến trình nền vừa được khởi chạy; biến `MONITOR_PID` giúp dùng lại đúng PID đó.

Kết quả của lần chạy kiểm thử script:

```text
[1] 1969
PID: 1969
```

> PID thay đổi ở mỗi lần chạy. Khi thực hành, sử dụng PID do chính lệnh `$!` hoặc `pgrep` trả về, không dùng lại PID đã ghi nhận ở trên.

## 4. Kiểm tra tiến trình

Tìm PID bằng một trong các lệnh sau:

```bash
echo "$MONITOR_PID"
pgrep -af '[l]oop-monitor.sh'
ps aux | grep '[l]oop-monitor.sh'
```

Dùng mẫu `[l]oop-monitor.sh` để dòng lệnh `grep` không tự xuất hiện trong kết quả. Kết quả minh họa:

```text
student  1969  0.0  0.0  7376  3328 pts/0  S  20:45  0:00 /bin/bash ./loop-monitor.sh
```

Có thể theo dõi thêm bằng `top -p "$MONITOR_PID"` hoặc mở `htop` rồi tìm PID trên.

## 5. Kiểm tra log

Đợi ít nhất 10 giây rồi kiểm tra:

```bash
tail -n 10 /tmp/monitor.log
```

Kết quả minh họa cho thấy log được ghi đều sau mỗi 5 giây:

```text
System time: Tue Oct  6 20:39:07 SEAST 2026
System time: Tue Oct  6 20:39:12 SEAST 2026
System time: Tue Oct  6 20:39:17 SEAST 2026
```

Sau khi đóng Terminal/SSH, đăng nhập lại và chạy hai lệnh sau để xác nhận tiến trình vẫn hoạt động:

```bash
pgrep -af '[l]oop-monitor.sh'
tail -n 10 /tmp/monitor.log
```

## 6. Dừng tiến trình bằng tín hiệu

Ưu tiên gửi `SIGTERM` (số 15) để tiến trình có cơ hội kết thúc an toàn:

```bash
kill -15 "$MONITOR_PID"
```

Nếu đã đăng nhập lại và không còn biến `MONITOR_PID`, tìm lại PID rồi dừng:

```bash
MONITOR_PID=$(pgrep -f '[l]oop-monitor.sh' | head -n 1)
kill -15 "$MONITOR_PID"
```

Do script có chu kỳ `sleep 5`, chờ tối đa khoảng 5 giây rồi kiểm tra:

```bash
ps -p "$MONITOR_PID"
pgrep -af '[l]oop-monitor.sh'
```

Khi tiến trình đã dừng, `ps -p` chỉ còn dòng tiêu đề hoặc không có kết quả; `pgrep` không trả về dòng nào.

Chỉ khi tiến trình vẫn tồn tại sau khi đã chờ và kiểm tra mới dùng `SIGKILL` (số 9):

```bash
kill -9 "$MONITOR_PID"
```

`SIGKILL` buộc tiến trình dừng ngay và không thể được script bắt hoặc xử lý, vì vậy không nên dùng ngay từ đầu.

## 7. Kết luận

Script chạy nền bằng `nohup` tiếp tục ghi thời gian vào `/tmp/monitor.log` khi phiên SSH kết thúc. Tiến trình được xác định bằng `$!`, `pgrep` hoặc `ps` và được dừng an toàn bằng `kill -15`; `kill -9` chỉ là phương án cuối cùng.
