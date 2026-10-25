# Phần 4: Kịch bản Thực hành Lab End-to-End (CI/CD + DevSecOps + Ansible)

Tài liệu này hướng dẫn chi tiết cách vận hành, kiểm thử hệ thống **CI/CD Pipeline tích hợp DevSecOps** và triển khai tự động ứng dụng **Cloud Note App (3-Tier)** lên **Target Linux VM** bằng **Ansible**, kèm theo kịch bản quay Video Demo thuyết trình trong 5 - 7 phút.

---

## 1. Sơ đồ Kiến trúc & Luồng Hoạt động End-to-End

```text
+----------------------------------------------------------------------------------------------------------------+
|                                   HỆ THỐNG LAB CI/CD + DEVSECOPS + ANSIBLE                                     |
+----------------------------------------------------------------------------------------------------------------+

  [Developer]
       │
       │ 1. git push (Code + Dockerfile + Ansible Roles + Pipeline)
       ▼
+-------------------------------------------------------------------+
| GIT REPOSITORY & CI/CD PLATFORM (GitLab CI/CD hoặc Jenkins)       |
|  - Quản lý mã nguồn & Masked Secrets (SSH_PRIVATE_KEY, TOKENS)    |
+-------------------------------------------------------------------+
       │
       │ 2. Webhook kích hoạt CI/CD Runner / Agent
       ▼
+---------------------------------------------------------------------------------------------------+
| CI/CD RUNNER & ANSIBLE CONTROL NODE                                                               |
|                                                                                                   |
|  [Stage 1: Secret & SAST]  ──>  [Stage 2: Build Image]  ──>  [Stage 3: Container Scan]            |
|   - Gitleaks (Quét Secret)       - Build Frontend Image       - Trivy Scan (Security Gate)        |
|   - Bandit (SAST) & Pytest       - Build Backend Image        - Push lên Container Registry       |
|                                                                        │                          |
|                                                                        ▼                          |
|                                                        [Stage 4: Ansible Automated Deploy]        |
|                                                         - Chạy ansible-playbook deploy.yml        |
+---------------------------------------------------------------------------------------------------+
                                                                         │
                                            3. Pull Image                │ 4. SSH (Port 22 - Agentless)
                                               từ Registry               ▼
+-----------------------------------+                     +-----------------------------------------+
| CONTAINER REGISTRY                | <================== | TARGET LINUX VM (Ubuntu Server)         |
| (Docker Hub / GitLab Registry)    |                     |  - Role 1: Cài đặt Docker & Compose V2  |
|  - cloud-note-frontend:2.0.x      |                     |  - Role 2: Render .env & compose.yaml   |
|  - cloud-note-backend:2.0.x       |                     |  - Khởi chạy 3 Containers + Healthcheck |
+-----------------------------------+                     +-----------------------------------------+
```

---

## 2. Cấu trúc Thư mục Thực hành (`week4/`)

```text
week4/
├── README.md                                          # Kiến thức trọng tâm Tuần 4 (CI/CD, Ansible, DevSecOps)
├── Simulation.md                                      # Hướng dẫn vận hành Lab & Kịch bản quay Video Demo
├── AI_Helper/
│   ├── README.md                                      # Phân rã yêu cầu & Lộ trình học tập
│   └── REPORT.md                                      # Báo cáo chuyên sâu toàn diện
└── lab-cicd-ansible/                                  # Mã nguồn thực hành Lab End-to-End
    ├── .gitlab-ci.yml                                 # Định nghĩa Pipeline cho GitLab CI/CD
    ├── Jenkinsfile                                    # Định nghĩa Declarative Pipeline cho Jenkins
    ├── .gitleaks.toml                                 # Cấu hình quy tắc quét lộ lọt Secret (Gitleaks)
    ├── .trivyignore                                   # Danh sách quản lý ngoại lệ CVE (Trivy)
    └── ansible/                                       # Cấu hình trên Ansible Control Node
        ├── ansible.cfg                                # Tối ưu hóa SSH, Pipelining & Privilege Escalation
        ├── deploy.yml                                 # Master Playbook điều phối triển khai
        ├── inventory/
        │   └── hosts.ini                              # Danh sách địa chỉ IP của Target Linux VM
        └── roles/
            ├── docker_setup/                          # Role 1: Tự động cài đặt Docker Engine & Compose V2
            │   └── tasks/main.yml
            └── cloud_note_deploy/                     # Role 2: Triển khai 3-Tier App & Kiểm tra Healthcheck
                ├── defaults/main.yml                  # Biến mặc định (Image tag, Port, DB config)
                ├── handlers/main.yml                  # Handler khởi động lại stack khi cấu hình đổi
                ├── tasks/main.yml                     # Các bước tạo file, pull image, verify /health
                └── templates/
                    ├── .env.j2                        # Jinja2 Template cho file biến môi trường bảo mật
                    └── compose.yaml.j2                # Jinja2 Template cho Docker Compose 3 Containers
```

---

## 3. Các bước Thiết lập và Vận hành Thực tế

### Bước 1: Chuẩn bị Kết nối SSH giữa Ansible Control Node và Target Linux VM
Trên máy đóng vai trò **Ansible Control Node** (hoặc CI/CD Runner):

```bash
# 1. Tạo cặp khóa SSH chuẩn Ed25519
ssh-keygen -t ed25519 -C "cicd-ansible@typ-training" -f ~/.ssh/id_ed25519 -N ""

# 2. Chép Public Key sang Target Linux VM (thay 192.168.56.105 bằng IP thực tế của VM)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.56.105

# 3. Trên Target Linux VM: Cấp quyền sudo không cần mật khẩu cho user ubuntu
ssh ubuntu@192.168.56.105 "sudo sh -c 'echo \"ubuntu ALL=(ALL) NOPASSWD: ALL\" > /etc/sudoers.d/ubuntu && chmod 0440 /etc/sudoers.d/ubuntu'"
```

### Bước 2: Kiểm tra kết nối bằng Ansible Ad-hoc Command
Di chuyển vào thư mục `lab-cicd-ansible/ansible` và kiểm tra kết nối tới Target VM:

```bash
cd lab-cicd-ansible/ansible

# Kiểm tra kết nối bằng module ping
ansible all -i inventory/hosts.ini -m ping

# Kiểm tra thông tin OS và thời gian hoạt động của Target VM
ansible all -i inventory/hosts.ini -a "uname -a"
```

### Bước 3: Chạy thử Kiểm tra Bảo mật (DevSecOps) cục bộ trước khi Push
Bạn có thể chạy thử từng bước của Pipeline ngay trên máy để kiểm chứng các chốt chặn **Security Gate**:

```bash
# 1. Quét lộ lọt Secret bằng Gitleaks
docker run --rm -v "$(pwd):/path" zricethezav/gitleaks:latest \
  detect --source="/path" --config="/path/taind_b24dccn510_training_gd1/week4/lab-cicd-ansible/.gitleaks.toml" --verbose --redact

# 2. Quét lỗ hổng mã nguồn tĩnh SAST cho Backend Python bằng Bandit
bandit -r ../week3/app/backend -ll -ii

# 3. Quét lỗ hổng CVE trên Docker Image bằng Trivy (Security Gate)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image --severity CRITICAL --ignore-unfixed --exit-code 1 skyboy12/cloud-note-backend:2.0.0
```

### Bước 4: Kiểm thử và Thực thi Ansible Playbook Triển khai lên Target VM
Trước khi để CI/CD tự động chạy, kiểm tra cú pháp và chạy chế độ mô phỏng (`--check --diff`):

```bash
cd lab-cicd-ansible/ansible

# 1. Kiểm tra cú pháp YAML của Playbook và các Roles
ansible-playbook -i inventory/hosts.ini deploy.yml --syntax-check

# 2. Chạy thử chế độ Dry-run (--check) và xem sự thay đổi file (--diff)
ansible-playbook -i inventory/hosts.ini deploy.yml --check --diff

# 3. Thực thi triển khai chính thức lên Target VM
ansible-playbook -i inventory/hosts.ini deploy.yml \
  -e "ansible_host=192.168.56.105" \
  -e "backend_image=skyboy12/cloud-note-backend:2.0.0" \
  -e "frontend_image=skyboy12/cloud-note-frontend:2.0.0"
```

### Bước 5: Cấu hình Secrets trên GitLab CI/CD hoặc Jenkins
Vào mục **Settings $\rightarrow$ CI/CD $\rightarrow$ Variables** (trên GitLab) hoặc **Manage Jenkins $\rightarrow$ Credentials** (trên Jenkins) và khai báo các biến bảo mật sau:

| Tên Biến (Key) | Giá trị mẫu / Mô tả | Thuộc tính Bảo mật |
| :--- | :--- | :--- |
| `REGISTRY_USER` | `skyboy12` (Username Docker Hub / Registry) | Protected |
| `REGISTRY_PASSWORD` | Personal Access Token (PAT) của Docker Hub | **Masked** & Protected |
| `SSH_PRIVATE_KEY` | Nội dung file `~/.ssh/id_ed25519` để SSH vào VM | **File / Masked** & Protected |
| `TARGET_VM_IP` | `192.168.56.105` (IP của Target Linux VM) | Protected |
| `DB_PASSWORD` | `SecurePassword2026@` | **Masked** |
| `DB_ROOT_PASSWORD` | `RootAdminSecret2026@` | **Masked** |

---

## 4. Kịch bản Quay Video Demo Thuyết trình (5 - 7 Phút)

Để bài thuyết trình ngắn gọn, súc tích và kiểm soát tốt thời gian, bạn có thể quay video demo theo 4 phân cảnh sau:

1. **Phân cảnh 1 (1 phút) - Giới thiệu Kiến trúc & Mã nguồn:**
   - Mở sơ đồ kiến trúc và thư mục `lab-cicd-ansible/`.
   - Giải thích nhanh 4 Stages trong `.gitlab-ci.yml` (hoặc `Jenkinsfile`) và cấu trúc 2 Roles (`docker_setup`, `cloud_note_deploy`) trong thư mục `ansible/roles/`.
2. **Phân cảnh 2 (1.5 phút) - Kích hoạt CI/CD Pipeline & Quan sát Security Gate:**
   - Thực hiện sửa nhẹ giao diện hoặc thêm 1 commit mới $\rightarrow$ gõ `git push origin main`.
   - Mở giao diện Pipeline đang chạy:
     - Show log của **Gitleaks** (không phát hiện rò rỉ khóa bí mật) và **Bandit + Automated Unit Test** báo xanh.
     - Show log của **Trivy** quét 2 Docker Images (`frontend` & `backend`) với kết quả `0 CRITICAL vulnerabilities`.
3. **Phân cảnh 3 (1.5 phút) - Quan sát Ansible Tự động Triển khai lên Target VM:**
   - Mở log của Stage `deploy_to_vm_ansible`:
     - Chỉ rõ tính **Idempotency (Lũy đẳng)**: Các bước cài đặt Docker báo `ok` (màu xanh lá) nếu VM đã có Docker, chỉ bước cập nhật `.env`/`compose.yaml` và kéo Image mới báo `changed` (màu vàng).
     - Chỉ vào bước cuối của Playbook: Task `Chờ và xác thực Healthcheck của ứng dụng` trả về `status: 200` (`{"database": "connected", "status": "healthy"}`).
4. **Phân cảnh 4 (1 phút) - Kiểm chứng Ứng dụng trên Target Linux VM:**
   - Mở trình duyệt truy cập địa chỉ IP của máy ảo `http://<TARGET_VM_IP>:8080`.
   - Thêm một ghi chú mới trên giao diện Web để chứng minh cả 3 tầng (**Frontend Nginx $\rightarrow$ Backend Flask API $\rightarrow$ Database MySQL 8.0**) đã hoạt động trơn tru.

