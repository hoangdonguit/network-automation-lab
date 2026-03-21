# Lab môn NT542.Q21 - Lập trình kịch bản tự động hóa cho quản trị và bảo mật mạng
**Giảng viên:** Cô Trần Thị Dung

**Thực hiện bởi:** Hoàng Xuân Đồng - 23520297

---

## 🎯 Mục tiêu Lab
Dự án này sử dụng Ansible để tự động hóa quy trình cấu hình và sao lưu cho các thiết bị mạng Cisco IOS. Cụ thể bao gồm 2 yêu cầu chính:
- **Task 1 (Provisioning):** Tự động đẩy cấu hình bảo mật cơ bản (bao gồm thiết lập `enable secret`, mật khẩu `line console`, và kích hoạt `service password-encryption`) cho 2 Router.
- **Task 2 (Backup):** Tự động kết nối, thu thập cấu hình đang chạy (`running-config`) của cả 2 Router và lưu trữ thành file `.txt` trên máy quản trị cục bộ.

---

## 📐 Sơ đồ Mạng (Topology) & Môi trường giả lập
- **Control Node (Máy quản trị):** Máy Host Ubuntu 24.04 (Cài đặt Ansible & Git).
- **Managed Nodes (Thiết bị chịu quản lý):** 02 Router Cisco c7200 (R1, R2).
- **Network Subnet:** `172.17.0.0/24`.
<img width="721" height="435" alt="Sơ đồ Topology Lab" src="https://github.com/user-attachments/assets/bfe8ccfb-f4ad-41e5-9064-55545ba65549" />
### 💡 Lựa chọn Kiến trúc: Ứng dụng Cloud Node
Thay vì triển khai một máy ảo Ubuntu lồng bên trong GNS3 làm Ansible Node, dự án sử dụng đối tượng **Cloud (NAT/Bridge)** để nối trực tiếp sơ đồ mạng ảo ra card mạng (`docker0`/`virbr0`) của máy Host. 
**Ưu điểm của giải pháp này:**
1. **Tối ưu tài nguyên:** Tránh hao phí RAM và CPU cho việc chạy thêm một máy ảo Control Node dư thừa.
2. **Hiệu suất vận hành:** Cho phép Kỹ sư lập trình, kiểm thử playbook trực tiếp trên IDE (VS Code) và thao tác với Git ngay trên môi trường máy Host một cách liền mạch.
3. **Mô phỏng thực tế:** Phản ánh đúng mô hình quản trị doanh nghiệp, trong đó Kỹ sư từ máy trạm Local thực hiện remote (SSH) vào hệ thống mạng lõi để vận hành.

---

## ⚙️ Yêu cầu hệ thống (Prerequisites)
Để khởi chạy kịch bản tự động hóa này, hệ thống cần đáp ứng:
1. **Hạ tầng mạng:** R1 và R2 đã được cấu hình địa chỉ IP, hostname và kích hoạt dịch vụ SSH (đã khởi tạo RSA key, username/password).
2. **Control Node:** Đã cài đặt Ansible và thư viện hỗ trợ giao thức mạng.
   ```bash
   sudo apt update && sudo apt install ansible git -y
   pip3 install ansible-pylibssh --break-system-packages
