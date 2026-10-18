# Phần 7: Thực hành triển khai ứng dụng đa Container bằng Docker Compose

Tài liệu này hướng dẫn chi tiết quy trình xây dựng, cấu hình, vận hành và kiểm thử ứng dụng **Cloud Note App** (Web Python Flask kết nối hệ quản trị cơ sở dữ liệu **MySQL 8.0**) sử dụng **Docker Compose V2**, Docker Volume bền vững và mạng nội bộ cách ly theo chuẩn Production.

---

## 1. Kiến trúc ứng dụng

```text
                               +-------------------------------------------------------+
                               |                 DOCKER HOST (Linux/PC)                |
                               |                                                       |
  Browser / Client (Host)      |   +-------------------+       +-------------------+   |
  http://localhost:8080 ------>|-->| notes_web_service |       |  notes_db_service |   |
                               |   |   (Flask App)     |       |    (MySQL 8.0)    |   |
                               |   +---------+---------+       +---------+---------+   |
                               |             |                           |             |
                               |             +-------------+-------------+             |
                               |                           |                           |
                               |            MẠNG NỘI BỘ (Embedded DNS: 'db')           |
                               |              [week3_notes_network]                    |
                               |                                         |             |
                               |                                         v             |
                               |                         +-------------------------+   |
                               |                         | DOCKER PERSISTENT VOLUME|   |
                               |                         | [week3_mysql_notes_vol] |   |
                               |                         +-------------------------+   |
                               +-------------------------------------------------------+
```

### Thành phần hệ thống:
1. **Web Service (`notes_web_service`):**
   - Xây dựng bằng **Python 3.11 Flask**.
   - Giao diện người dùng trực quan hiển thị thông tin hệ thống, phiên bản MySQL và cho phép thêm, xem, xóa các ghi chú.
   - Đóng gói bằng **Multi-stage build Dockerfile**, chạy dưới quyền **Non-root user (`appuser`)** nhằm hạn chế rủi ro bảo mật.
   - Tích hợp endpoint `/health` để Docker giám sát sức khỏe định kỳ (`HEALTHCHECK`).
2. **Database Service (`notes_db_service`):**
   - Sử dụng **MySQL 8.0** chính thức.
   - Cấu hình kiểm tra sức khỏe tự động qua lệnh `mysqladmin ping` trước khi cho phép Web kết nối (`condition: service_healthy`).
   - Dữ liệu được lưu trữ độc lập trên **Docker Named Volume (`week3_mysql_notes_volume`)** để đảm bảo không bị mất khi container tắt hoặc bị xóa.
3. **Cấu hình & Mạng:**
   - Mọi thông tin nhạy cảm (mật khẩu, tên database, user) được tách biệt trong file **`.env`**.
   - Mạng nội bộ **`week3_notes_network` (Bridge)**: Tự động phân giải tên service `db` thông qua Embedded DNS Server của Docker (`127.0.0.11`).

---

## 2. Cấu trúc thư mục mã nguồn

```text
week3/
├── app/
│   ├── app.py                  # Mã nguồn ứng dụng web Flask & xử lý CRUD MySQL
│   ├── requirements.txt        # Danh sách thư viện Python cần cài đặt
│   ├── Dockerfile              # Chỉ thị Multi-stage build tối ưu cho Web container
│   ├── .dockerignore           # Loại bỏ các file thừa khỏi Build context
│   ├── .env                    # File biến môi trường chứa cấu hình & mật khẩu bí mật
│   ├── compose.yaml            # File điều phối hệ thống đa container Docker Compose V2
│   ├── compose.override.yaml   # Cấu hình ghi đè tự động dành cho môi trường Development
│   └── run_standalone.sh       # Shell script chạy ứng dụng trực tiếp không qua Docker
├── Simulation.md               # Kịch bản thực hành và hướng dẫn vận hành chi tiết
└── README.md                   # Tài liệu lý thuyết và báo cáo tổng hợp tuần 3
```

---

## 3. Các bước vận hành thực tế

### Bước 1: Di chuyển vào thư mục ứng dụng & Kiểm tra tính hợp lệ của cấu hình
Trước khi khởi động, di chuyển vào thư mục `app` và kiểm tra xem Docker Compose đã phân giải các biến môi trường từ file `.env` chính xác chưa:

```bash
cd app
docker compose config
```

### Bước 2: Xây dựng và khởi chạy hệ thống ở chế độ nền
Lệnh sau sẽ tự động build image cho web app từ `Dockerfile`, tải image `mysql:8.0`, tạo volume, tạo network và khởi động các container:

```bash
docker compose up -d --build
```

### Bước 3: Quan sát tiến trình khởi động & Trạng thái Healthcheck
Xem danh sách container và kiểm tra trạng thái sức khỏe:

```bash
docker compose ps
```

*Nhận xét:*
- Ban đầu, container `notes_db_service` ở trạng thái `health: starting`.
- Nhờ cấu hình `depends_on: { db: { condition: service_healthy } }`, container `notes_web_service` sẽ kiên nhẫn chờ đến khi MySQL hoàn tất khởi tạo và chuyển sang `healthy` mới bắt đầu chạy, ngăn ngừa hoàn toàn lỗi `Can't connect to MySQL server`.

### Bước 4: Kiểm tra Logs hệ thống
```bash
# Xem log toàn bộ hệ thống theo thời gian thực
docker compose logs -f

# Hoặc chỉ xem log của service web
docker compose logs -f web
```

### Bước 5: Kiểm tra ứng dụng trên trình duyệt hoặc CLI
1. Mở trình duyệt và truy cập: **`http://localhost:8080`**
2. Hoặc kiểm tra nhanh bằng lệnh `curl`:
```bash
curl http://localhost:8080
```
3. Thử tạo vài ghi chú trên giao diện web (ví dụ: `Ghi chú thử nghiệm 1`, `Kiểm tra Docker Volume`).

---

## 4. Kiểm chứng tính bền vững của Docker Volume (Persistence Test)

Đây là bài kiểm tra quan trọng nhất để chứng minh dữ liệu cơ sở dữ liệu không bị mất mát khi vòng đời của container kết thúc:

### Bước 1: Dừng và XÓA HOÀN TOÀN các container
```bash
docker compose down
```
Lệnh này sẽ dừng và xóa container `notes_web_service` và `notes_db_service`, nhưng **giữ nguyên Docker Volume**.

### Bước 2: Kiểm tra Volume vẫn tồn tại trên máy Host
```bash
docker volume ls | grep week3_mysql_notes_volume
```

### Bước 3: Khởi động lại hệ thống
```bash
docker compose up -d
```

### Bước 4: Kiểm tra lại dữ liệu
Truy cập lại `http://localhost:8080`: Toàn bộ các ghi chú đã tạo từ trước vẫn còn nguyên vẹn 100%!

---

## 5. Mẹo mở rộng trong thực tế

### 1. Xóa sạch toàn bộ hệ thống bao gồm cả Volume
Khi muốn làm sạch toàn bộ dữ liệu để cài đặt lại từ đầu:
```bash
docker compose down -v
```

### 2. Nhân bản (Scale) container Web
Khi lượng truy cập tăng cao, bạn có thể tăng số lượng container phục vụ web:
```bash
docker compose up -d --scale web=3
```
*(Lưu ý: Không cố định host port trong `compose.yaml` khi dùng scale, hoặc sử dụng thêm Nginx Reverse Proxy làm Load Balancer phía trước).*

---

## 6. Khởi chạy ứng dụng Standalone (Không qua Docker & Không cần Database bên ngoài)

Dành cho các tình huống kiểm thử nhanh, demo logic ứng dụng trực tiếp trên máy chủ / máy ảo mà không muốn cài đặt Docker hay khởi chạy hệ quản trị MySQL bên ngoài:

### Cách thực hiện bằng Shell Script:
Tệp kịch bản `run_standalone.sh` đã được thiết kế sẵn:
1. Tự động kiểm tra phiên bản `python3` hoặc `python` trên hệ thống.
2. Tự khởi tạo môi trường ảo Python (`venv`) độc lập.
3. Cài đặt các dependencies cần thiết từ `requirements.txt`.
4. Kích hoạt chế độ `STANDALONE_MODE=1` (sử dụng SQLite lưu trữ cục bộ tại `app/notes_local.db`).
5. Khởi chạy ứng dụng trên cổng mặc định `8080` (hoặc cổng được cấu hình qua biến môi trường `PORT`).

```bash
# Di chuyển vào thư mục app
cd app

# Cấp quyền thực thi và chạy script
chmod +x run_standalone.sh
./run_standalone.sh
```

- Mở trình duyệt và truy cập: **`http://localhost:8080`**
- Giao diện sẽ tự động hiển thị chế độ hoạt động: **`Standalone Mode (Host Direct / No-Docker)`** cùng cơ sở dữ liệu **`SQLite (Local Embedded)`**, hỗ trợ đầy đủ các thao tác Thêm, Xem và Xóa ghi chú mà hoàn toàn không phụ thuộc vào Docker hay MySQL daemon ngoài.

