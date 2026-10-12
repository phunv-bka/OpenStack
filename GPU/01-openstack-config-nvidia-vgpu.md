# Cấu hình vGPU NVIDIA trên OpenStack

💡 Đây chỉ là tài liệu tham khảo. Các thông số kỹ thuật hay lỗi có thể phát sinh.

---
# 1. Tổng quan môi trường

| Thành phần | Thông tin |
|---|---|
| OpenStack | 2025.1 |
| Deployment | Kolla-Ansible |
| Compute OS | Ubuntu 24.04 |
| GPU | NVIDIA L40 |
| GPU `vendor_id` | `10de` |
| GPU `product_id` | `26b5` |
| GPU Driver | NVIDIA vGPU Driver 595.91.04 |
| Compute Host | `openstack-compute-vgpu` |

> **Phạm vi:** Tài liệu này tập trung vào cấu hình NVIDIA L40 vGPU trên compute node và tích hợp tài nguyên vGPU vào Nova.

---

# 2. Kiến trúc triển khai

Quy trình tổng thể:

```text
NVIDIA L40 GPU on OpenStack
      │
      ├── NVIDIA vGPU Host Driver
      │
      ├── IOMMU
      │
      ├── SR-IOV
      │
      ├── Virtual Functions (VF)
      │      ├── virtfn0 → PCI 0000:2a:00.4
      │      ├── virtfn1 → PCI 0000:2a:00.5
      │      └── virtfn2 → PCI 0000:2a:00.6
      │      └── virtfnx → PCI 0000:xx:xx.x      
      ├── NVIDIA vGPU Type
      │      └── L40-16A → ID 888
      │
      └── Nova PCI configuration
             │
             ├── device_spec
             ├── alias
             └── PciPassthroughFilter
                    │
                    └── OpenStack Flavor
                           │
                           └── VM
```
---

# 3. Chuẩn bị Compute Host

> Thực hiện trên compute node có GPU NVIDIA L40.

## 3.1. Cập nhật hệ điều hành

```bash
apt update -y
apt upgrade -y
```

## 3.2. Cài đặt các package cần thiết

```bash
apt-get install linux-source -y
apt-get install linux-headers-generic -y
update-pciids
```

Cài đặt GCC/build tools:

```bash
apt install build-essential -y
```

---

# 4. Enable IOMMU

Chỉnh sửa:

```bash
/etc/default/grub
```

- Host sử dụng CPU Intel CPU

  ```bash
  GRUB_CMDLINE_LINUX="intel_iommu=on"
  ```

- Host sử dụng CPU AMD CPU

  ```bash
  GRUB_CMDLINE_LINUX="amd_iommu=on"
  ```

Sau đó rebuild GRUB:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

# 5. Disable Nouveau

Tạo file:

```bash
cat <<EOF | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
```

Khởi tạo lại initramfs:

```bash
sudo update-initramfs -u
```

Reboot compute host:

```bash
init 6
```

---

# 6. Kiểm tra KVM/IOMMU

Sau khi host khởi động lại, thực hiện kiểm tra trong container `nova_libvirt`:

```bash
virt-host-validate
```

Kết quả mong muốn:

```text
QEMU: Checking if IOMMU is enabled by kernel : PASS
```

Nếu kiểm tra IOMMU trả về `PASS`, host đã nhận diện IOMMU đúng.

---

# 7. Cài NVIDIA vGPU Host Driver

## 7.1. Download driver

Download NVIDIA vGPU Host Driver dành cho Ubuntu KVM từ NVIDIA Licensing Portal:

```text
https://ui.licensing.nvidia.com/software
```

## 7.2. Cài đặt driver

Ví dụ:

```bash
cd ~/GPU_dirver/Host_Drivers/
```

Kiểm tra file:

```bash
ll
```

Cài đặt:

```bash
apt install ./nvidia-vgpu-ubuntu-595_595.91.04_amd64.deb -y
```

Reboot:

```bash
init 6
```

Kiểm tra NVIDIA GPU:

```bash
nvidia-smi
```

---

# 8. Kiểm tra NVIDIA vGPU Host

## 8.1. Kiểm tra kernel modules

```bash
lsmod | grep nvidia
```

Các module quan trọng cần xuất hiện gồm:

```text
nvidia_vgpu_vfio
nvidia
vfio_pci_core
mdev
vfio
```

Ví dụ:

```text
nvidia_vgpu_vfio   122880     12
nvidia             14819328   4 nvidia_vgpu_vfio
vfio_pci_core      94208      1 nvidia_vgpu_vfio
mdev               24576      1 nvidia_vgpu_vfio
vfio               69632      3 vfio_pci_core,nvidia_vgpu_vfio,vfio_iommu_type1
ecc                45056      1 nvidia
irqbypass          12288      3 vfio_pci_core,nvidia_vgpu_vfio,kvm
```

## 8.2. Kiểm tra GPU bằng `nvidia-smi vgpu`

```bash
nvidia-smi vgpu
```

Ví dụ:

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 595.91.04              Driver Version: 595.91.04                 |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|     vGPU ID  Name               | VM ID  VM Name               | vGPU-Util  |
|=================================+==============================+============|
|  0  NVIDIA L40                  | 00000000:2A:00.0             | 0%         |
+---------------------------------+------------------------------+------------+
|  1  NVIDIA L40                  | 00000000:99:00.0             | 0%         |
+---------------------------------+------------------------------+------------+
```

---

# 9. Xác định PCI Address của GPU

Kiểm tra IOMMU group:

```bash
ls -ld /sys/kernel/iommu_groups/*/devices/*2a:00.?/
ls -ld /sys/kernel/iommu_groups/*/devices/*99:00.?/
```

Ta sẽ lấy ra được PCI address của card GPU:
```text
0000:2a:00.0
0000:99:00.0
```

Các PCI address này sẽ được sử dụng khi enable SR-IOV.

---

# 10. Enable SR-IOV cho NVIDIA L40

Sử dụng `sriov-manage`:

```bash
/usr/lib/nvidia/sriov-manage -e 0000:2a:00.0
/usr/lib/nvidia/sriov-manage -e 0000:99:00.0
```

Sau bước này, NVIDIA GPU sẽ tạo các Virtual Function (VF).

---

# 11. Tự động Enable SR-IOV sau khi Reboot

Để không phải chạy `sriov-manage` thủ công sau mỗi lần reboot, tạo script systemd.

## 11.1. Tạo script

```bash
sudo tee /usr/local/sbin/nvidia-sriov-enable.sh > /dev/null <<'EOF'
#!/bin/bash

/usr/lib/nvidia/sriov-manage -e 0000:2a:00.0
/usr/lib/nvidia/sriov-manage -e 0000:99:00.0
EOF
```

Cấp quyền:

```bash
sudo chmod +x /usr/local/sbin/nvidia-sriov-enable.sh
```

## 11.2. Tạo systemd service

```bash
sudo tee /etc/systemd/system/nvidia-sriov-enable.service > /dev/null <<'EOF'
[Unit]
Description=Enable NVIDIA SR-IOV Virtual Functions
After=nvidia-persistenced.service
Wants=nvidia-persistenced.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/nvidia-sriov-enable.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

## 11.3. Enable service

```bash
sudo systemctl daemon-reload
sudo systemctl enable nvidia-sriov-enable.service
```

Kiểm tra:

```bash
systemctl status nvidia-sriov-enable.service
```

---

# 12. Xác định các vGPU Type

NVIDIA cung cấp danh sách vGPU type thông qua:

```bash
cat /sys/bus/pci/devices/0000:2a:00.0/virtfn0/nvidia/creatable_vgpu_types
```

Ví dụ: danh sách trên NVIDIA L40:

| ID | vGPU Type |
|---:|---|
| 869 | NVIDIA L40-1B |
| 870 | NVIDIA L40-2B |
| 871 | NVIDIA L40-1Q |
| 872 | NVIDIA L40-2Q |
| 873 | NVIDIA L40-3Q |
| 874 | NVIDIA L40-4Q |
| 875 | NVIDIA L40-6Q |
| 876 | NVIDIA L40-8Q |
| 877 | NVIDIA L40-12Q |
| 878 | NVIDIA L40-16Q |
| 879 | NVIDIA L40-24Q |
| 880 | NVIDIA L40-48Q |
| 881 | NVIDIA L40-1A |
| 882 | NVIDIA L40-2A |
| 883 | NVIDIA L40-3A |
| 884 | NVIDIA L40-4A |
| 885 | NVIDIA L40-6A |
| 886 | NVIDIA L40-8A |
| 887 | NVIDIA L40-12A |
| 888 | NVIDIA L40-16A |
| 889 | NVIDIA L40-24A |
| 890 | NVIDIA L40-48A |
| 2163 | NVIDIA L40-3B |

> **Lưu ý:** `ID` trong bảng trên là mã được sử dụng khi cấu hình `current_vgpu_type`.

---

# 13. Cấu hình vGPU Type cho VF

Trong ví dụ này sử dụng:

```text
vGPU Type : NVIDIA L40-16A
vGPU ID   : 888
```

Kiểm tra các VF:

```bash
cd /sys/bus/pci/devices/0000:2a:00.0/
ls -l | grep virtfn
```

Ví dụ:

```text
virtfn0 -> ../0000:2a:00.4
virtfn1 -> ../0000:2a:00.5
virtfn2 -> ../0000:2a:00.6
...
```

## 13.1. Mapping VF → PCI Address
Với profile NVIDIA L40-16A thì cấu hình tối đa 3 vGPU nên ví dụ này sẽ lấy ra 3 VF để cấu hình:

Xem thêm thông tin về giới hạn số vGPU cấu hình trên 1 card GPU: https://docs.nvidia.com/vgpu/20.0/grid-vgpu-user-guide/virtual-gpu-types-grid-reference.html#vgpu-types-nvidia-l40

| VF | PCI Address |
|---|---|
| `virtfn0` | `0000:2a:00.4` |
| `virtfn1` | `0000:2a:00.5` |
| `virtfn2` | `0000:2a:00.6` |

## 13.2. Gán vGPU Type
Thêm ID của vGPU type NVIDIA L40-16A để khai báo cho các VF:

```bash
cd /sys/bus/pci/devices/0000:2a:00.0/
echo 888 > virtfn0/nvidia/current_vgpu_type 2>/dev/null
echo 888 > virtfn1/nvidia/current_vgpu_type 2>/dev/null
echo 888 > virtfn2/nvidia/current_vgpu_type 2>/dev/null
```

Sau đó các VF tương ứng có thể được Nova sử dụng để cấp vGPU.

**Note:** Để Remove vGPU Type, thực hiện remove id khỏi VF bằng cách thay bằng `0`

```bash
echo 0 > virtfn0/nvidia/current_vgpu_type 2>/dev/null
```

Tương tự đối với các VF còn lại.

---

# 14. Cấu hình Nova Compute

> Thực hiện trên Compute Host cung cấp GPU.

File:

```text
/etc/kolla/nova-compute/nova.conf
```
```ini
[pci]

# Profile L40-16A
device_spec = {"vendor_id":"10de","product_id":"26b5","address":{"domain":"0000","bus":"2a","slot":"00","function":"[4-6]"},"resource_class":"CUSTOM_NVIDIA_888","managed":"no"}
alias = {"name":"L40-16A","vendor_id":"10de","product_id":"26b5","device_type":"type-VF","resource_class":"CUSTOM_NVIDIA_888"}

# Profile L40-24A
device_spec = {"vendor_id":"10de","product_id":"26b5","address":{"domain":"0000","bus":"99","slot":"00","function":"[4-5]"},"resource_class":"CUSTOM_NVIDIA_889","managed":"no"}
alias = {"name":"L40-24A","vendor_id":"10de","product_id":"26b5","device_type":"type-VF","resource_class":"CUSTOM_NVIDIA_889"}
```

### Ý nghĩa các tham số

| Tham số | Ý nghĩa |
|---|---|
| `vendor_id` | Vendor ID của NVIDIA, lấy từ `lspci -nn \| grep -i nvidia` |
| `product_id` | Product ID của VF, lấy từ `lspci -nn \| grep -i nvidia` |
| `device_type` | Sử dụng `type-VF` vì GPU sử dụng SR-IOV |
| `address` | PCI address pattern của VF cần cung cấp cho Nova |
| `name` | Tên alias của PCI device, ví dụ `L40-16A` |
| `resource_class` | Resource class dùng để mapping device với alias |
| `managed` | Đặt `no` theo cấu hình trong note |

Tham khảo Nova PCI configuration: 

- [pci.device_spec](https://docs.openstack.org/nova/latest/configuration/config.html#pci.device_spec)
- [pci.alias](https://docs.openstack.org/nova/latest/configuration/config.html#pci.alias)

---

# 15. Restart Nova Compute

Sau khi thay đổi `nova.conf`:

```bash
docker restart nova_compute
```

---

# 16. Cấu hình Nova Controller

> Thực hiện trên các Controller node.

## 16.1. Cấu hình `nova-api`

Bổ sung cấu hình nova-api:
```text
/etc/kolla/nova-api/nova.conf
```
```ini
[filter_scheduler]
enabled_filters = ...,PciPassthroughFilter

[pci]
alias = {"vendor_id":"10de","product_id":"26b5","name":"L40-16A","device_type":"type-VF","resource_class":"CUSTOM_NVIDIA_888"}
alias = {"vendor_id":"10de","product_id":"26b5","name":"L40-24A","device_type":"type-VF","resource_class":"CUSTOM_NVIDIA_889"}
```

## 16.2. Cấu hình `nova-scheduler`

File:

```text
/etc/kolla/nova-scheduler/nova.conf
```

Bổ sung:

```ini
[filter_scheduler]
enabled_filters = ...,PciPassthroughFilter
```

## 16.3. Restart service

```bash
docker restart nova_api nova_scheduler
```

ℹ️ Nếu triển khai nhiều Controller node, cần đảm bảo cấu hình được đồng bộ trên các node tương ứng.

---

# 17. Tạo Flavor cho VM vGPU

> Thực hiện trên Controller node.

Trong cấu hình hiện tại, mỗi VM được tạo với **1 vGPU** thông qua flavor.

## 17.1. Flavor L40-16A

```bash
openstack flavor create \
  --vcpus 16 \
  --ram 16384 \
  --disk 0 \
  --property pci_passthrough:alias="L40-16A:1" \
  1vgpuL40-16A.16c16r
```

## 17.2. Flavor L40-24A

```bash
openstack flavor create \
  --vcpus 16 \
  --ram 16384 \
  --disk 0 \
  --property pci_passthrough:alias="L40-24A:1" \
  1vgpuL40-24A.16c16r
```

Sau khi tạo flavor, sử dụng flavor tương ứng khi tạo VM.

---

# 18. Kiểm tra vGPU trên Compute Host

Sau khi VM được tạo, kiểm tra trên compute node:

```bash
nvidia-smi vgpu
```

Ví dụ:

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 595.91.04              Driver Version: 595.91.04                 |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|     vGPU ID  Name               | VM ID  VM Name               | vGPU-Util  |
|=================================+==============================+============|
|  0  NVIDIA L40                  | 00000000:2A:00.0             | 0%         |
|     3251634284  NVIDIA L40-16A  | acf1... instance-00000ad2    | 0%         |
+---------------------------------+------------------------------+------------+
|  1  NVIDIA L40                  | 00000000:99:00.0             | 0%         |
+---------------------------------+------------------------------+------------+
```

Trong ví dụ trên, GPU `0000:2a:00.0` đang cung cấp 3 vGPU `L40-16A`.

---

# 19. Kiểm tra GPU bên trong VM

SSH hoặc console vào VM và thực hiện:

```bash
lspci | grep NVIDIA
```

Nếu PCI device của NVIDIA xuất hiện, VM đã được Nova gán GPU/VF.

---

# Tài liệu tham khảo
- https://docs.openstack.org/nova/latest/admin/virtual-gpu.html
- https://docs.openstack.org/nova/latest/admin/pci-passthrough.html#configure-nova-compute
- https://docs.openstack.org/nova/latest/configuration/config.html#pci.device_spec
- https://docs.openstack.org/nova/latest/configuration/config.html#pci.alias
- https://docs.nvidia.com/vgpu/20.0/grid-vgpu-user-guide/virtual-gpu-types-grid-reference.html