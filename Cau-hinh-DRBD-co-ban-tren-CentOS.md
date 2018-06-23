# Hướng dẫn cài đặt và cấu hình DRBD trên CentOS 7

### Menu

- [1. Chuẩn bị](#1)
- [2. Các bước tiến hành](#2)
	- [2.1 Cài đặt DRBD](#2.1)
	- [2.2 Cấu hình DRBD](#2.2)
	- [2.3 Khởi động trên mỗi node](#2.3)
	- [2.4 Bật và kích hoạt DRBD daemon](#2.4)
	- [2.5 Kích hoạt trên node chính](#2.5)
	- [2.6 Tạo mà mount file system DRBD](#2.6)
	- [2.7 Test hoạt động replicate](#2.7)
- [3. Tham khảo](#3)

<a name="1"></a>

## 1. Chuẩn bị

- 2 server sử dụng OS CentOS
- 2 ổ cứng có cùng dung lượng được gắn vào các node
- Cấu hình hostname cho các server
- Mở port 7788 trên các server

Cụ thể:

**Node 1**

```
OS: CentOS 7 64 bit
Device: /dev/sdb - 8GB
Hostname: node1
IP: 192.168.100.196
Gateway: 192.168.100.1
Network: 192.168.100.0/24
```

**Node 2**

```
OS: CentOS 7 64 bit
Device: /dev/sdb - 8GB
Hostname: node2
IP: 192.168.100.197
Gateway: 192.168.100.1
Network: 192.168.100.0/24
```

#### Mô hình

<img src="https://camo.githubusercontent.com/bf6d9f67c22c5f4944f5fa334789e63ca5d5c64c/687474703a2f2f696d6167652e70726e747363722e636f6d2f696d6167652f37653965316132366537656634623630393361663633383434393135303532652e706e67" width=75% />

<a name="2"></a>

## 2. Các bước tiến hành

<a name="2.1"></a>

### 2.1 Cài đặt DRBD

- Trước khi cài đặt, chúng ta phải cấu hình hostname cho mỗi node và ghi chúng vào `hosts`

```
[root@node1 ~] hostnamectl set-hostname node1
```

```
[root@node2 ~] hostnamectl set-hostname node2
```

- Ghi thêm vào `hosts` của mỗi server

```
vi /etc/hosts
```

```
[...]
192.168.100.196 node1
192.168.100.197 node2
```

- Tiếp theo, chúng ta cài đặt DRBD trên cả 2 server. Đầu tiên, chúng ta thêm repos của DRBD và key 

```
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
pm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
``` 

- Trước khi cài, chúng ta kiểm tra phiên bản mới nhất của DRBD.

```
yum info *drbd* | grep Name

Name        : drbd84-utils
Name        : drbd84-utils-sysvinit
Name        : kmod-drbd84
```

Sau khi chạy lệnh trên, chúng ta thấy phiên bản hiện tại là **drbd84**

- Chúng ta tiếp tục bước cài đặt DRBD trên cả 2 server.

```
yum -y install drbd84-utils kmod-drbd84
```

**Chú ý**: Nếu key trên bị lỗi, hãy làm bước sau để import các key có sẵn ở `/etc/pki/rpm-gpg/` và làm lại bước cài đặt trên.

```
rpm --import /etc/pki/rpm-gpg/*
```

- Kích hoạt module DRBD trên cả 2 server

```
modprobe drbd
```

- Kiểm tra lại xem DRBD đã hoạt động:

```
lsmod | grep drbd

drbd                  405309  0
libcrc32c              12644  2 xfs,drbd
```

<a name="2.2"></a>

### 2.2 Cấu hình DRBD

File cấu hình chính của DRBD nằm là ` /etc/drbd.conf`. File này gọi lại các file cấu hình được khai báo trong phần nội dung. Các file `.res` dùng để khai báo tài nguyên trên mỗi server mà DRBD sử dụng.

Chúng tạo một file có tên `testdata1.res` với nội dung như sau:

```
vi /etc/drbd.d/testdata1.res
```

```
resource testdata1 {
protocol C;          
on node1 {
                device /dev/drbd0;
                disk /dev/sdb;
                address 192.168.100.196:7788;
                meta-disk internal;
        }
on node2 {
                device /dev/drbd0;
                disk /dev/sdb;
                address 192.168.100.197:7788;
                meta-disk internal;
        }
} 
```
**Giải thích:**

- `resource testdata1`: Tên của resource
- `Protocol C`: Các resource được cấu hình để synchronous replication. Chi tiết [tại đây](README.md)
- `node1`, `node2`: Danh sách các node và các tùy chọn bên trong
- `device /dev/drbd0`: Xác định thiết bị logic được DRBD sử dụng (Nên đặt giống nhau ở trên 2 server)
- `disk /dev/sdb`: Xác định thiết bị vật lý dùng để tạo ra thiết bị logic bên trên, và không nhất thiết phải cùng trên trên 2 server.
- `address 192.168.100.197:7788`: Xác định địa chỉ IP và Port của mỗi server
- `meta-disk internal`: Cho phép sử dụng Meta-data mức nội bộ

Sao chép file cấu hình sang server 2:

```
[root@node1 ~] scp /etc/drbd.d/testdata1.res node2:/etc/drbd.d/
```

<a name="2.3"></a>

### 2.3 Khởi động trên mỗi node

Trên server 1:

```
[root@node1 ~] drbdadm create-md testdata1
```

Trên server 2:

```
[root@node2 ~] drbdadm create-md testdata1
```

Khi thấy kết quả hiển thị như sau báo hiệu đã cấu hình thành công:

```
  --==  Thank you for participating in the global usage survey  ==--
The server's response is:
you are the 10680th user to install this version
initializing activity log
NOT initializing bitmap
Writing meta data...
New drbd meta data block successfully created.
success
```

<a name="2.4"></a>

### 2.4 Bật và kích hoạt DRBD daemon

Ở trên 2 server, chúng ta bật và cho DRBD khởi động cùng hệ thống

```
systemctl start drbd
systemctl enable drbd
```

<a name="2.5"></a>
### 2.5 Kích hoạt trên node chính

Tôi chọn node chính là node1, chúng ta cũng có thể chọn node2 làm node chính bằng cách chạy lệnh này lên node2.

```
[root@node1 ~] drbdadm primary testdata1 --force
```
Kiểm tra trạng thái:

```
[root@node1 ~]# cat /proc/drbd
version: 8.4.7-1 (api:1/proto:86-101)
GIT-hash: 3a6a769340ef93b1ba2792c6461250790795db49 build by phil@Build64R7, 2016-01-12 14:29:40
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:1048508 nr:0 dw:0 dr:1049236 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
```

Hoặc dùng lệnh:

```
[root@node1 ~]# drbd-overview
 0:testdata1/0  Connected Primary/Secondary UpToDate/UpToDate
```

**Chú ý**: Chúng ta kiểm tra liên tục bằng lệnh trên và khi nào lệnh trả về kết quả tương tự hoặc có chứa nội dung `Connected Primary/Secondary` thì mới có thể chuyển sang bước 2.6.

<a name="2.6"></a>
### 2.6 Tạo mà mount file system DRBD

Tạo một file system và mount, ghi dữ liệu lên nó. Các bước thực hiện trên node chính - node mà bạn đã kích hoạt ở bước 2.5

```
[root@node1 ~]# mkfs.ext3 /dev/drbd0
[root@node1 ~]# mount /dev/drbd0 /mnt
[root@node1 ~]# touch /mnt/testfile
[root@node1 ~]# ll /mnt/
total 16
drwx------ 2 root root 1384 Oct  12 08:29 lost+found
-rw-r--r-- 1 root root    0 Oct  12 08:31 testfile
```

<a name="2.7"></a>
### 2.7 Test hoạt động replicate trên server 2

Chúng ta chuyển primary node sang node2 để kiểm tra dữ liệu có được replicate

Unmount file system trên node1

```
[root@node1 ~]# umount /mnt
```

Chuyển sang chế độ `secondary node` cho node1

```
[root@node1 ~] drbdadm secondary testdata1
```

Trên node2, chúng ta kích hoạt chế độ primary

```
[root@node2 ~] drbdadm primary testdata1
```

Mount và kiểm tra dữ liệu bên trong

```
[root@node2 ~]# mount /dev/drbd0 /mnt
[root@node2 ~]# ll /mnt/
total 16
drwx------ 2 root root 1384 Oct  12 08:29 lost+found
-rw-r--r-- 1 root root    0 Oct  12 08:31 testfile
```

Kết quả trên cho ta thấy, dữ liệu đã được replicate sang node2.

<a name="3"></a>
## 3. Tham khảo

- http://www.learnitguide.net/2016/07/how-to-install-and-configure-drbd-on-linux.html

**Đừng bỏ lỡ:**

- [Cách khắc phục lỗi cơ bản của DRBD](Split-Brain.md)
- [Tích hợp giữa DRBD với Pacemaker + Corosync để HA cho Web Server](https://github.com/hoangdh/ghichep-HA/blob/master/Pacemaker_Corosync/2.Huong-dan-Pacemaker-Corosync-cho-Web-DRBD-CentOS.md)