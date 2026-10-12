# BÀI 1 – BÁO CÁO KỸ THUẬT: CONTAINER VS VIRTUAL MACHINE

## 1. Bối cảnh và mục tiêu

Trong quá trình hiện đại hóa hạ tầng, doanh nghiệp thường cân nhắc chuyển cách triển khai ứng dụng từ các VPS (máy chủ ảo) riêng lẻ sang Docker Container. Cần phân biệt: VPS thường là một máy ảo được cấp phát trên máy chủ vật lý; Docker là công nghệ đóng gói và chạy ứng dụng. Docker có thể chạy **ngay trên VPS**, vì vậy chuyển sang Docker không nhất thiết là loại bỏ VPS mà là thay đổi đơn vị triển khai, cách quản lý phụ thuộc và vòng đời ứng dụng.

Máy ảo sử dụng hypervisor để ảo hóa phần cứng. Mỗi VM có hệ điều hành khách và kernel riêng, cùng thư viện, dịch vụ và ứng dụng. Container sử dụng container engine/runtime để khởi chạy tiến trình được cô lập; các container Linux trên cùng Linux host dùng chung kernel của host. Trên Windows chạy Docker Desktop, Linux container thường chạy bên trong môi trường Linux VM/WSL2, nên vẫn có một lớp ảo hóa bên dưới.

## 2. Bảng so sánh VM và Container

| Tiêu chí | Virtual Machine (VM/VPS) | Docker Container |
|---|---|---|
| Dung lượng Image | Thường tính bằng GB do chứa hệ điều hành khách và các thành phần hệ thống | Thường nhỏ hơn, có thể từ vài chục đến hàng trăm MB; image ứng dụng phức tạp vẫn có thể lớn |
| Thời gian khởi động | Thường từ hàng chục giây đến vài phút vì cần khởi động OS | Thường từ dưới một giây đến vài giây cho tiến trình; thời gian sẵn sàng phụ thuộc ứng dụng |
| Hao phí OS | Mỗi VM có kernel và các tiến trình hệ thống riêng, tiêu tốn thêm RAM, CPU, lưu trữ | Chia sẻ kernel host, không cần OS khách đầy đủ cho mỗi container; giảm chi phí tài nguyên |
| Mức độ cô lập | Cô lập ở ranh giới phần cứng ảo, kernel riêng; thường mạnh hơn | Cô lập ở cấp tiến trình/kernel bằng namespaces, cgroups và các cơ chế bảo mật bổ sung; cùng chia sẻ kernel |
| Khả năng mở rộng | Tạo và cấu hình VM thường nặng hơn, nhưng phù hợp tải cần hệ điều hành riêng | Khởi tạo nhiều bản sao ứng dụng nhanh, dễ kết hợp CI/CD và hệ thống điều phối như Kubernetes |

Các nhận xét trên là xu hướng phổ biến, không phải cam kết tốc độ hoặc dung lượng cố định. Kết quả thực tế phụ thuộc base image, tài nguyên, hệ điều hành, ứng dụng và cấu hình hạ tầng.

## 3. Namespaces – cơ chế cô lập tài nguyên

Linux namespaces tạo ra các góc nhìn tài nguyên khác nhau cho từng nhóm tiến trình, giúp container hoạt động như một môi trường tương đối độc lập:

- **PID namespace (Process):** Tách không gian định danh tiến trình. Tiến trình bên trong container có thể nhìn thấy cây tiến trình riêng, thay vì toàn bộ tiến trình của host.
- **Network namespace:** Cung cấp ngăn xếp mạng riêng gồm interface, địa chỉ IP, bảng định tuyến và cổng. Các container có thể kết nối với nhau thông qua Docker bridge network và phân giải tên trên custom network.
- **Mount namespace:** Tách góc nhìn các điểm mount và hệ thống tệp. Container có filesystem riêng theo image; bind mount và volume có thể chia sẻ hoặc duy trì dữ liệu ngoài vòng đời container.

Ngoài ra, UTS namespace cô lập hostname, IPC namespace cô lập tài nguyên giao tiếp liên tiến trình, user namespace ánh xạ UID/GID để giảm đặc quyền, và cgroup namespace cô lập góc nhìn về phân cấp cgroups. Namespaces không phải hàng rào bảo mật tuyệt đối: cấu hình đặc quyền, quyền truy cập socket Docker, capabilities và lỗ hổng kernel vẫn có thể tạo rủi ro.

## 4. Control Groups (cgroups) – quản lý tài nguyên

**cgroups** là cơ chế của Linux để theo dõi, phân nhóm và kiểm soát mức sử dụng tài nguyên của tiến trình. Khác với namespaces (chủ yếu cô lập góc nhìn), cgroups giúp ngăn một container sử dụng quá mức tài nguyên và ảnh hưởng đến ứng dụng khác.

- **CPU:** Có thể giới hạn thời lượng CPU mà container được sử dụng, ví dụ `docker run --cpus="1.0" ...`.
- **RAM:** Có thể đặt giới hạn bộ nhớ, ví dụ `docker run --memory="512m" ...`. Khi vượt giới hạn, container có thể bị kết thúc do thiếu bộ nhớ (OOM).
- **Các tài nguyên khác:** Tùy phiên bản kernel và cấu hình, cgroups có thể quản lý I/O, số lượng tiến trình và các loại tài nguyên khác.

Docker có thể sử dụng cgroup v1 hoặc v2 tùy môi trường; `Cgroup Driver` cho biết cơ chế quản lý cgroup mà Docker sử dụng, ví dụ `systemd` hoặc `cgroupfs`. Cần đọc giá trị thực tế từ `docker info`.

## 5. Thông số Docker thực tế

Chạy các lệnh sau trên máy thực hành:

```bash
docker version
docker info
docker info | grep -E "Server Version:|Cgroup Driver:|Containers:|Images:"
```

Điền thông số dựa trên kết quả thực tế (không tự suy đoán):

| Thông số | Giá trị thực tế |
|---|---|
| Server Version | **[Điền từ docker info]** |
| Cgroup Driver | **[Điền từ docker info]** |
| Containers | **[Điền từ docker info]** |
| Images | **[Điền từ docker info]** |

**Minh chứng:** Chụp màn hình terminal hiển thị `docker info` và lưu tại `images/docker-info.png`. Trên Windows Docker Desktop, thực hiện trong Git Bash/PowerShell với tài khoản người dùng thông thường, không cần gõ `sudo`.

## 6. Nhận định cho doanh nghiệp và kết luận

Docker giúp chuẩn hóa môi trường triển khai bằng image, giảm tình trạng “chạy trên máy lập trình nhưng lỗi trên server”, rút ngắn thời gian phát hành và rollback, hỗ trợ nhân bản dịch vụ và tận dụng tài nguyên hiệu quả hơn. Với ứng dụng backend, việc tách API và database thành các container, dùng custom network cho kết nối nội bộ và named volume cho dữ liệu bền vững là một ví dụ thực tế.

Tuy nhiên, Docker không thay thế mọi chức năng của VPS/VM: doanh nghiệp vẫn cần host để chạy container, giám sát, cập nhật kernel, bảo vệ bí mật, sao lưu dữ liệu, quản lý network và kiểm soát quyền. Các hệ thống yêu cầu kernel riêng, cô lập bảo mật mạnh hoặc nhiều hệ điều hành khác nhau có thể tiếp tục dùng VM. Vì vậy, lựa chọn hợp lý thường là **chạy các ứng dụng đóng gói bằng Docker trên hạ tầng VM/VPS**, thay vì xem VM và Container là hai giải pháp loại trừ nhau.
