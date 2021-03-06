# Hướng dẫn đóng image Ubuntu 14.04 với cloud-init và QEMU Guest Agent (không dùng LVM)

## Chú ý:

- Hướng dẫn này dành cho các image không sử dụng LVM
- Sử dụng công cụ virt-manager hoặc web-virt để kết nối tới console máy ảo
- OS cài đặt KVM là Ubuntu 14.04
- Phiên bản OpenStack sử dụng là Queens
- Hướng dẫn bao gồm 2 phần chính: thực hiện trên máy ảo cài OS và thực hiện trên KVM Host

----------------------

## Bước 1: Tạo máy ảo bằng kvm

Bạn có thể dử dụng virt-manager hoặc virt-install để tạo máy ảo

Ở đây mình sử dụng virt-install

``` sh
qemu-img create -f qcow2 /tmp/ubuntu14.qcow2 10G

virt-install --virt-type kvm --name ubuntu14 --ram 1024 \
  --cdrom=/var/lib/libvirt/images/ubuntu-14.04.4-server-amd64.iso \
  --disk /tmp/ubuntu14.qcow2,format=qcow2 \
  --network bridge=br0 \
  --graphics vnc,listen=0.0.0.0 --noautoconsole \
  --os-type=linux --os-variant=ubuntu14.04
```

**Một số lưu ý trong quá trình cài đặt**

- Đối với hostname, các bạn có thể đặt mặc định bởi ta dùng cloud-init để set sau.
- Đối với cấu hình partion, để standard cloud-init với 1 phân vùng root (/) để máy ảo có thể tự resize theo flavor mới.

<img src="http://i.imgur.com/hI7aW14.png">

- Đối với phần `software selection`, ta lựa chọn `OpenSSH server`

<img src="http://i.imgur.com/oLB72zc.png">

- Install GRUB boot loader

- Sau khi cài đặt xong, chọn `Continue` để reboot máy ảo.
Lưu ý: Có một số trường hợp đối với ubuntu14.04, máy ảo sẽ không reboot kể cả khi nó báo là sẽ reboot

## Bước 2 : Tắt máy ảo, xử lí trên KVM host

- Chỉnh sửa file `.xml` của máy ảo, bổ sung thêm channel trong <devices> (để máy host giao tiếp với máy ảo sử dụng qemu-guest-agent), sau đó save lại

`virsh edit ubuntu14`

với `ubuntu14` là tên máy ảo

``` sh
...
<devices>
 <channel type='unix'>
      <target type='virtio' name='org.qemu.guest_agent.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
 </channel>
</devices>
...
```


## Bước 3: Cài các dịch vụ cần thiết

Bật máy ảo lên, truy cập vào máy ảo. Lưu ý với lần đầu boot, bạn phải sử dụng tài khoản tạo trong quá trình cài os, chuyển đổi nó sang tài khoản root để sử dụng.

Cấu hình cho phép login root và xóa user `ubuntu` chỉnh `vi /etc/ssh/sshd_config`
```sh
PermitRootLogin yes
```

Đặt passwd cho root
```sh
sudo su 
passwd
Enter new UNIX password: <root_passwd>
Retype new UNIX password: <root_passwd>
```

Restart sshd
```sh
service ssh restart
```

Logout login bằng user `root` và xóa user `ubuntu`
```sh
userdel ubuntu
rm -rf /home/ubuntu
```

**Cài đặt cloud-init, cloud-utils và cloud-initramfs-growroot**

`apt-get install cloud-utils cloud-initramfs-growroot cloud-init -y`

## Bước 4: Cấu hình để instance nhận metadata từ datasource

`dpkg-reconfigure cloud-init`

Sau khi màn hình mở ra, lựa chọn `EC2` và `OpenStack`

## Bước 5: Cấu hình user nhận ssh keys

Thay đổi file `/etc/cloud/cloud.cfg` để chỉ định user nhận ssh keys khi truyền vào, mặc định là `root`

``` sh
sed -i 's/disable_root: false/disable_root: true/g' /etc/cloud/cloud.cfg
sed -i 's/name: ubuntu/name: root/g' /etc/cloud/cloud.cfg
```

## Bước 6: Xóa bỏ thông tin của địa chỉ MAC

Xóa nội dung file `/lib/udev/rules.d/75-persistent-net-generator.rules` và `/etc/udev/rules.d/70-persistent-net.rules` (file này được gen bởi file trước) bằng các sử dụng `:%d`  trong `vi`.

Bạn cũng có thể thay thế file trên bằng 1 file rỗng. Lưu ý: không được xóa bỏ hoàn toàn file mà chỉ xóa nội dung.

## Bước 7: Cấu hình để instance báo log ra console

Thay đổi `GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8"` trong file `/etc/default/grub`.

Sau đó nhập lệnh `update-grub` để lưu lại.

## Bước 8: Cài đặt netplug

- Để sau khi boot máy ảo, có thể nhận đủ các NIC gắn vào:

``` sh
apt-get install netplug -y
wget https://raw.githubusercontent.com/longsube/Netplug-config/master/netplug
```

- Đưa file netplug vào thư mục /etc/netplug

``` sh
mv netplug /etc/netplug/netplug
chmod +x /etc/netplug/netplug
```

## Bước 9: Disable default config route

Comment dòng `link-local 169.254.0.0` trong `/etc/networks`

## Bước 10: Cài đặt qemu-guest-agent


Chú ý: qemu-guest-agent là một daemon chạy trong máy ảo, giúp quản lý và hỗ trợ máy ảo khi cần (có thể cân nhắc việc cài thành phần này lên máy ảo)

Để có thể thay đổi password máy ảo thì phiên bản qemu-guest-agent phải >= 2.5.0

``` sh
apt-get install software-properties-common -y
add-apt-repository cloud-archive:mitaka -y
apt-get update
apt-get install qemu-guest-agent -y
```

Kiểm tra phiên bản qemu-ga bằng lệnh:

`qemu-ga --version`

Kết quả:

`QEMU Guest Agent 2.5.0`

Kiểm tra dịch vụ

`service qemu-guest-agent status`

Kết quả:

`* qemu-ga is running`

## Bước 11: Cấu hình card mạng tự động active khi hệ thống boot-up

``` sh
vim /etc/network/interfaces

auto lo
iface lo inet loopback
auto eth0
iface eth0 inet dhcp
```

## Bước 12: Tắt máy ảo

`init 0`


## Bước 13: Clean up image

`virt-sysprep -d ubuntu14`

## Bước 14: Undefine libvirt domain

`virsh undefine ubuntu14`

## Bước 15: Giảm kích thước máy ảo

`virt-sparsify --compress /tmp/ubuntu14.qcow2 /root/ubuntu14.img`

**Lưu ý:**

Nếu img bạn sử dụng đang ở định dạng raw thì bạn cần thêm tùy chọn `--convert qcow2` để giảm kích thước image.

## Bước 16: Upload image lên glance

- Di chuyển image tới máy CTL, sử dụng câu lệnh sau

``` sh
glance image-create --name Ubuntu14-64bit-2018 \
--disk-format qcow2 \
--container-format bare \
--file Ubuntu14-64bit-2018.img \
--visibility=public \
--property hw_qemu_guest_agent=yes \
--progress
```

- Kiểm tra xem image đã upload thành công chưa, kiểm tra metadata của image đã có `hw_qemu_guest_agent` hay chưa.

<img src="https://i.imgur.com/RxeuiFr.png">

<img src="https://i.imgur.com/whh1wh0.png">


**Link tham khảo:**

https://github.com/hocchudong/Image_Create/blob/master/docs/Ubuntu14.04_noLVM%2Bqemu_ga.md

https://github.com/thaonguyenvan/meditech-thuctap/blob/master/ThaoNV/Tim%20hieu%20OpenStack/docs/image-create/Ubuntu14-04-cloudinit-noLVM.md
