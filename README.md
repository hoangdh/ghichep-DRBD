# Tìm hiểu về DRBD

### Menu

- [1. DRBD là gì?](#1)
- [2. Quá trình hoạt động cơ bản của DRBD](#2)
- [3. Các chế độ replication của DRBD](#3)
	- [3.1 Protocol A](#3.1)
	- [3.2 Protocol B](#3.2)
	- [3.3 Protocol C](#3.3)    
- [4. Cấu trúc của DRBD](#4)
- [5. Tham khảo](#5)

## 1. DRBD là gì? <a name="1"></a>

`DRBD` viết tắt của **Distributed Replicated Block Device**, là một tiện ích sử dụng để nâng cao tính sẵn sàng của hệ thống. Nó là được xây dựng trên nền ứng dụng mã nguồn mở để đơn giản hóa việc chia sẻ các dữ liệu trong hệ thống lưu trữ với nhau qua đường truyền mạng. Chúng ta có thể hiểu nôm na rằng đây là RAID-1 sử dụng các giao tiếp mạng để trao đổi dữ liệu cho nhau.

Về tổng quan, DRBD gồm 2 server cung cấp 2 tài nguyên lưu trữ độc lập, không liên quan gì với nhau. Trong một thời điểm, một server sẽ được cấu hình làm node primary đọc và ghi dữ liệu; node còn lại là secondary làm nhiệm vụ đồng bộ dữ liệu từ node primary để đảm bảo tính đồng nhất dữ liệu 2 node.

## 2. Quá trình hoạt động cơ bản của DRBD <a name="2"></a>

Nhìn chung, các dịch vụ đồng bộ dữ liệu hoạt động ở chế độ Active - Passive. Ở chế độ này, node chính (primary) sẽ lắng nghe toàn bộ các thao tác (đọc, ghi) của người dùng. Node phụ (secondary) sẽ được kích hoạt thành node chính khi một giải pháp cluster nào đó phát hiện node chính down. Việc ghi dữ liệu sẽ được xảy ra đồng thời trên cả 2 node. DRBD hỗ trợ 2 kiểu ghi dữ liệu là fully synchronous and asynchronous.

<img width=150% src="http://i1363.photobucket.com/albums/r714/HoangLove9z/DRDB%20Operations_zpsa5lx9veo.jpg" />

DRBD cũng có thể hỗ trợ chế độ Active - Active, điều đó có nghĩa là việc ghi dữ liệu sẽ đồng thời xảy ra trên 2 node. Mode này được dựa trên một hệ thống chia sẻ tập tin, chẳng hạn như Global File System (GFS) hoặc các phiên bản Oracle Cluster File System 2 (OCFS2), trong đó bao gồm khả năng phân phối và quản lý file. 

## 3. Các chế độ replication của DRBD <a name="3"></a>

DRBD hỗ trợ 3 chế độ replication, cụ thể như sau:

### 3.1 Protocol A <a name="3.1"></a>

- Giao thức đồng bộ không đồng thời. Các thao tác ghi dữ liệu trên node chính sẽ được thực thi đến khi nào hoàn thành tác vụ, các gói tin replication được lưu trữ ở bộ đệm TCP. Trong trường hợp fail-over, dữ liệu này có thể bị mất.

### 3.2 Protocol B <a name="3.2"></a>

- Giao thức đồng bộ đồng thời trên RAM (semi-synchronous), các thao tác ghi được thực hiện trên node chính ngay khi có yêu cầu, các gói tin replication được gửi ngay khi node chính đã ghi xong.
Khi fail-over, dữ liệu sẽ bị mất.

### 3.3 Protocol C <a name="3.3"></a>

- Giao thức đồng bộ, dữ liệu được ghi hoàn thiện chỉ khi nào 2 node chính và phụ xác nhận là đã hoàn thành. Giao thức này đảm bảo tính an toàn dữ liệu và được sử dụng phổ biến khi cấu hình DRBD.

## 4. Cấu trúc của DRBD <a name="4"></a>

DRBD được chia làm 2 thành phần:

<img width=150% src="http://i1363.photobucket.com/albums/r714/HoangLove9z/DRDB%20Architecture_zpsbij78rwi.jpg" />

- Module trong kernel có nhiệm vụ thiết lập các space-user dùng để quản lý các disk DRBD, nó thực hiện các quyền điều khiển với các thiết bị virtual block (khi replicate dữ liệu local với sang máy remote). Giống như một virtual disk, DRBD cung cấp một mô hình linh loạt cho hầu hết các loại ứng dụng đều có thể sử dụng.
- Các module DRBD khởi tạo những chỉ dẫn các điểu khiển cơ bản được khai báo trong brbd.conf, ngoài ra còn phân định ra các thành phần được xác định bởi IP và Port

Một vài câu lệnh dùng để quản lý cài đặt DRBD:

- **DRBDadm**: Công cụ quản trị DRBD cao nhất
- **DRBDsetup**: Cấu hình DRBD nạp vào kernel
- **DRBDmeta**: Cho phép tạo, dump, khôi phục và chỉnh sửa cấu trúc của meta-data.

## 5. Tham khảo <a name="5"></a>

- http://www.learnitguide.net/2016/07/what-is-drbd-how-drbd-works-drbd.html
- Bonus: https://ngocdinhwanka.wordpress.com/2012/12/11/408/

**Đừng bỏ lỡ:**

- [Cài đặt và cấu hình DRBD trên CentOS7](Cau-hinh-DRBD-co-ban-tren-CentOS.md)
- [Cách khắc phục lỗi cơ bản của DRBD](Split-Brain.md)
- [Tích hợp giữa DRBD, Pacemaker + Corosync để HA cho Web Server](https://github.com/hoangdh/ghichep-HA/blob/master/Pacemaker_Corosync/2.Huong-dan-Pacemaker-Corosync-cho-Web-DRBD-CentOS.md)