# Dựng trạm duyệt bài, bàn giao media và báo cáo AI 24/7 qua Telegram trên VPS

> Bài gốc: [Thiên Mệnh — Mẹo Vặt AI](https://thienmenh.top/#page-meo-vat-ai/item/tram-duyet-bao-cao-ai-telegram-vps-24-7) · 26/11/2026

Khi chạy một đội nhiều tài khoản AI cùng lúc (viết nội dung, dựng video, đăng bài, kiểm duyệt), anh em rất dễ gặp cảnh AI đăng nhầm khi chưa duyệt, không truyền được file giữa các tài khoản, hoặc AI treo giữa chừng mà không ai hay. Tự mở máy tính cá nhân 24/7 vừa tốn điện vừa dễ rớt mạng.

Bài viết này chia sẻ trọn bộ quy trình dựng Trạm điều phối AI Telegram Hub v2.0 trên một VPS Ubuntu giá rẻ (~4–5 USD/tháng) bằng Docker Compose + Caddy HTTPS, kèm sẵn file tài liệu Markdown chứa AI Skill (SKILL.md) để anh em đưa cho Antigravity, Claude Code, Cursor hoặc Muse tự code trọn gói từ A–Z.

**Cẩm nang Quy trình VPS + AI Skill trọn gói (v2.0)** — Markdown (.md) · Quy trình 5 bước Docker/Caddy + SKILL.md + Prompt kết nối Agent. Tải quy trình & Skill tại [bài gốc trên thienmenh.top](https://thienmenh.top/#page-meo-vat-ai/item/tram-duyet-bao-cao-ai-telegram-vps-24-7).

## Mã mời Muse AI

Theo chương trình Mời bạn bè của Muse, người mới và người giới thiệu mỗi người nhận 1 tỷ token Muse khi người mới đổi mã mời hợp lệ trong vòng 48 giờ kể từ khi tham gia.

- M46O6V
- MLUFHL
- NAQQQU

Vào Cài đặt trong Muse và đổi một trong các mã trên trong thời hạn 48 giờ.

## 1. Kiến trúc trạm điều phối chạy 24/7 trên VPS giá rẻ

Toàn bộ trạm chỉ gồm một dịch vụ Python FastAPI gọn nhẹ (~60 MB RAM) kết hợp cơ sở dữ liệu SQLite chế độ WAL (không cần cài thêm PostgreSQL hay Redis) và đặt sau Caddy Reverse Proxy để tự động cấp, gia hạn chứng chỉ SSL HTTPS miễn phí từ Let's Encrypt.

Một VPS Ubuntu 22.04/24.04 cấu hình 1 vCPU, 1–2 GB RAM, 20 GB SSD là đủ gánh mượt cả đội AI chạy 24/7. Toàn bộ dữ liệu cơ sở dữ liệu, file bàn giao media và bản sao lưu hằng ngày đều được gắn ra Docker volume riêng nên cập nhật code không mất dữ liệu.

## 2. Bộ tính năng thực chiến trong bản v2.0

- Duyệt bài 2 cấp linh hoạt & xem trước ảnh: AI gửi thẳng bài lên đúng Topic Telegram của mình kèm nút Duyệt / Từ chối (tự hiển thị ảnh preview bằng sendPhoto nếu có link ảnh), hoặc gửi qua tài khoản Giám đốc QC duyệt trước, bài chưa đạt tự tạo phiếu sửa, ca khó đẩy lên Admin.
- Hỗ trợ đủ 3 loại tác vụ: Bài đăng mới (post), bình luận/trả lời có trích dẫn bài gốc & hạn dùng (reply) và bàn giao file (handoff).
- Kho Media Handoff 100 MB: Các AI upload ảnh/video/audio trực tiếp lên VPS lấy link tải có chữ ký (?token=...) bàn giao cho nhau và tự động xóa file hết hạn sau 7 ngày.
- Chống gửi trùng & tự hết hạn: Khử trùng lặp 2 lớp bằng khóa client_ref (30 ngày) và chuẩn hóa tiêu đề/link (60 phút); tự chuyển EXPIRED và gỡ nút duyệt trên Telegram khi bài reply quá hạn.
- Nhắn lệnh 2 chiều & cảnh báo thông minh: Nhắn thẳng trong Topic của AI nào là vào hộp thư chờ của AI đó; trong khung 08:00–22:00 giờ VN, bot tự báo khi có bài tồn quá 12 giờ hoặc có AI im lặng mất tín hiệu quá 3 giờ.
- Phân quyền API Key riêng & Mini Web Dashboard: Tách biệt Master Admin Key và Key phụ cho từng AI (chặn 403 nếu giả mạo tài khoản khác); quản trị tập trung tại /admin với bộ lọc, xem trước ảnh/video, upload media và biểu đồ sản lượng 7 ngày.

## 3. Quy trình 5 bước triển khai từ con số 0

1. Thuê VPS Ubuntu & trỏ tên miền: Tạo bản ghi A Record trỏ subdomain (ví dụ hub.yourdomain.com) về IPv4 của VPS và bật tường lửa UFW chỉ mở cổng 22, 80, 443. Nếu không muốn mở cổng inbound, file đính kèm có sẵn phương án B chạy Cloudflare Tunnel trong Docker Compose.
2. Tạo Bot Telegram & nhóm Forum Topics: Tạo bot qua @BotFather, vào Bot Settings → Group Privacy chọn Turn off để bot đọc được tin nhắn trong Topic. Tạo Telegram Group bật Topics, chia mỗi AI một Topic riêng và thêm bot làm Quản trị viên.
3. Cài Docker & sinh bộ khóa bảo mật: Cài Docker Engine bằng script chính thức, tạo thư mục /opt/ai-telegram-hub và dùng openssl sinh ngẫu nhiên Master Key cùng API Key riêng cho từng AI.
4. Nạp mã nguồn & cấu hình .env: Dùng AI Skill trong file đính kèm để tạo trọn bộ mã nguồn (server.py, db.py, telegram_client.py, config.py, dashboard.html, Dockerfile, docker-compose.yml, Caddyfile), điền token vào .env và khóa quyền chmod 600 .env.
5. Khởi chạy & gắn nhóm Telegram: Chạy docker compose up -d --build, kiểm tra /health trả về version 2.0.0. Vào phòng General gõ /bind, vào từng Topic gõ /bind <ma_agent> rồi mở https://hub.yourdomain.com/admin đăng nhập bằng Master Key.

## 4. Cách dùng file AI Skill để AI tự code trọn gói

Thay vì phải tự gõ hàng nghìn dòng code, bạn tải file Markdown đính kèm ở đầu bài rồi lưu Phần 3 thành tệp SKILL.md (trong thư mục skills của Antigravity, Claude Code hoặc Cursor), hoặc đính kèm thẳng cả file vào khung chat kèm câu prompt mẫu ở cuối bài.

Bản đặc tả Skill đã ghi chi tiết toàn bộ lược đồ 8 bảng SQLite, từng endpoint REST API, 6 vòng chạy nền, giao diện Mini Web Dashboard không phụ thuộc CDN ngoài và bộ kiểm thử tự động test_hub.py. Yêu cầu AI chạy pass toàn bộ test_hub.py trước khi đưa lên VPS.

## 5. Gắn prompt cho đội AI & lưu ý bảo mật

Trong Phần 4 của file đính kèm có sẵn 2 mẫu System Prompt hoàn chỉnh: một mẫu cho AI thành viên (gửi duyệt bài/reply, poll lệnh 2 chiều, báo done/fail, upload media bàn giao, nộp báo cáo cuối ngày) và một mẫu cho AI Giám đốc QC (duyệt hàng chờ QC, đẩy ca khó lên Admin, giao việc và xem feed tổng quan).

Khi vận hành thực tế, tuyệt đối không commit tệp .env lên Git công khai, luôn cấp Key phụ riêng trong HUB_ACC_KEYS cho từng AI để dễ thu hồi khi cần, và kiểm tra định kỳ các bản sao lưu SQLite tự động trong thư mục backups/.

## Lệnh cài Docker và tạo khóa bảo mật trên VPS Ubuntu

Chạy sau khi SSH vào VPS và đã bật UFW (mở cổng 22, 80, 443):

```
curl -fsSL https://get.docker.com | sh
mkdir -p /opt/ai-telegram-hub/{data,media_storage,backups}
cd /opt/ai-telegram-hub
echo "MASTER_KEY=$(openssl rand -hex 24)"
echo "KEY_LEAD=ak_lead_$(openssl rand -hex 16)"
echo "KEY_CONTENT=ak_content_$(openssl rand -hex 16)"
```

## Khởi chạy trạm 24/7 và kiểm tra HTTPS

Sau khi đặt mã nguồn, Caddyfile, docker-compose.yml và .env vào /opt/ai-telegram-hub:

```
chmod 600 .env
docker compose up -d --build
docker compose exec hub python test_hub.py
curl -s https://hub.yourdomain.com/health
```

## Nguồn tham khảo

- FastAPI · Tài liệu chính thức
- Telegram Bot API · sendPhoto, Topics và Inline Keyboard
- Caddy Server · Tự động cấp SSL HTTPS với Let's Encrypt
- Docker Compose · Tài liệu triển khai dịch vụ 24/7
- Cloudflare Tunnel · Phương án không mở port trên VPS

## Prompt mẫu

Hãy đọc kỹ tệp quy-trinh-va-skill-tram-duyet-ai-telegram-vps.md đính kèm (đặc biệt là Phần 3: AI Skill & Blueprint). Dựa đúng theo đặc tả v2.0 đó, hãy tạo trọn bộ mã nguồn hoàn chỉnh cho dự án Trạm duyệt bài, bàn giao media và báo cáo AI qua Telegram chạy Docker Compose + Caddy HTTPS trên VPS Ubuntu (gồm config.py, db.py, telegram_client.py, server.py, dashboard.html, test_hub.py, requirements.txt, Dockerfile, docker-compose.yml, Caddyfile, .env.example). Dùng tên tài khoản mẫu trung tính, không hardcode bất kỳ token hay thông tin cá nhân nào. Sau khi viết xong, hãy chạy kiểm tra cú pháp và chạy pass toàn bộ test_hub.py trước khi bàn giao.
