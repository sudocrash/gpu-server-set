# 0. 사전 작업

## 0.1. 우분투 VM 설치

### 1) (General)
- VM ID는 다른 VM들과 겹치지 않는 적당한 숫자로 Name 역시 본인이 원하는 이름으로 지정할 것(ex: GPUServer)

### 2) (OS)
- storage: Backup
- ISO image: ubuntu-22.04.3-desktop-amd64.iso
- Type; Linux
- Version: 6.x - 2.6 Kernel

### 3) (System) --> 매우 중요합니다!!!
- Graphic card: Default
- machine: q35 # 반드시 이렇게 설정합니다!
- BIOS: OVMF(UEFI) # 반드시 이렇게 설정합니다!
- EFI Storage: Hynix-P31
- **Pre-Enroll keys 옵션을 체크 해제합니다**
* 보안부팅을 해제하는 옵션으로 반드시 이렇게 설정하도록 합니다!
* VM 생성 후 hardware나 .conf에서
`efidisk0: Hynix-P31:vm-999-disk-0,efitype=4m,size=4M`
위와 같이 설정이 되었는지 다시 한 번 확인합니다. 절대로 Pre-Enroll keys와 관련된 변수가 존재해서는 안됩니다!
* 보안 부팅을 해제함으로써 이후 NVIDIA 드라이버 설치시 MOK 활성화를 우회하게 됩니다. 따라서 반드시 해당 설정이 올바르게 이루어졌는지 확인할 수 있도록 합니다!

### 4) (Disks)
- Storage: Hynix-P31
- Disk size: 256 # 말그대로 VM 파티션 크기를 조절하는 부분입니다. 원하는 용량으로 정하면 됩니다.
- 이외의 옵션은 기본값으로 둡니다.

### 5) (CPU)
- Cores: 40
- 40코어 기준으로 필요한 코어수를 직접 할당합니다.
- 이외의 옵션은 기본값으로 둡니다.

### 6) (Memory)
- Memory: 65536
- 64GB 기준으로 마찬가지로 필요한 램 용량을 직접 할당합니다.

### 7) (Network)
- 모두 기본값으로 둡니다.

### 8) 1)~7)의 사항들이 모두 만족되었는지 확인합니다. 특히 3)의 System 설정을 다시 한 번 정확하게 확인할 수 있도록 합니다!!!

## 0.2. VM 부팅 및 openssh 설치

### 1) 0.1.에서 생성한 VM을 console에서 start합니다(VNC).

### 2) 우분투를 설치합니다.

### 3) 부팅한 후 다음과 같이 네트워크를 설정합니다.
(IPv4 - Manual)
Address: 10.191.65.XXX
netmask: 255.255.248.0
Gateway: 10.191.64.1
DNS: 8.8.8.8

### 4) 네트워크에 연결되었는지 확인합니다.

### 5) 터미널을 켜고 아래 명령어를 입력합니다.
sudo apt update
sudo apt install openssh-server

### 6) 아래 명령어를 입력해 ssh가 실행중인지 확인합니다
sudo systemctl status ssh

## 0.3. 컴퓨터에서 putty로 접속
### 1) putty에서 10.191.65.XXX로 접속합니다.
### 2) 아이디, 비번 입력 후 아래 명령어를 실행한 뒤 proxmox상에서 VM이 꺼지는지 확인합니다.
```bash
sudo poweroff
```
____________________

# 1. (node) shell GPU passtrough 작업
## 1) `nano /etc/default/grub` 후 다음과 같이 `GRUB_CMDLINE_LINUX_DEFAULT=` 부분을 변경합니다
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
```
## 2) 1)을 저장한 뒤 `update-grub`을 실행합니다

## 3) 다음을 실행합니다.
```bash
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" > /etc/modules
```

## 4) 다음을 실행하여 GPU의 PCI ID를 확인합니다.
```bash
lspci -nn | grep -i nvidia   
```

<출력>
```bash
2d:00.0 VGA compatible controller [10de:2231]  
2d:00.1 Audio device [10de:1aef]  
```
## 5) 다음을 실행합니다.
```bash
echo "options vfio-pci ids=10de:2231,10de:1aef" > /etc/modprobe.d/vfio.conf
echo -e "blacklist nouveau\nblacklist nvidia\nblacklist nvidiafb" > /etc/modprobe.d/blacklist.conf
update-initramfs -u -k all
echo 1 > /sys/bus/pci/devices/0000:2d:00.0/remove
echo 1 > /sys/bus/pci/rescan
```
## 6) nano /root/fix_gpu_pass.sh 실행 후 다음을 아래에 두 줄을 추가합니다.
```bash
echo 1 > /sys/bus/pci/devices/0000:2d:00.0/remove
echo 1 > /sys/bus/pci/rescan
```

## 7) 6)을 저장한 뒤, 아래를 실행합니다.
```bash
chmod +x /root/fix_gpu_pass.sh
```

## 8) crontab -e 실행 후 다음과 다음을 맨 아래에 추가합니다.
```bash
@reboot /root/fix_gpu_pass.sh
```
## 9) nano /etc/pve/qemu-server/XXX.conf 실행 후(VM ID에 따라 XXX를 수정) 다음을 추가합니다.
```bash
cpu: host,hidden=1
hostpci0: 2d:00.0,pcie=1,x-vga=1
hostpci1: 2d:00.1
```

## 10) reboot 합니다.

____________________

# 2. VM NVIDIA 드라이버 설치

## 1) VM을 start 합니다.

## 2) putty로 VM에 접속합니다.

## 3) 다음을 실행합니다. 참고로 서버에 설치되어 있는 RTX A5000의 경우 nvidia-driver-575가 적합합니다.
```bash
sudo apt install nvidia-driver-575
```
- 만약 해당 명령어 실행 후 MOK 설정 등의 requirements가 나타나는 경우 앞선 과정 중 문제가 있다는 뜻입니다(보안 부팅 해제를 안하는 등...). 따라서 VM을 삭제하시고 처음부터 다시 위의 과정들을 거치도록 합니다...

## 4) 설치가 완료되면 다음을 실행해 재부팅합니다.
```bash
sudo reboot
```

## 5) 재부팅 후 다시 putty로 VM에 접속합니다.

## 6) 다음을 실행하여 드라이버가 올바르게 설치되어 있는지 확인합니다.
```bash
nvidia-smi # GPU 인식 여부
watch -n 1 nvidia-smi # GPU 실시간 사용률
```

____________________

- 위의 과정들을 거치며 어떠한 지점에서라도 중간에 문제가 발생했다면 마음을 비우고 처음부터 과정들을 하나하나 꼼꼼하게 따라가며 실행해봅니다.
- MOK을 우회하는 것은 현재 물리 키보드와 마우스가 Host에 연결되어버리는 관계로 부팅시 MOK 인증을 할 수 없기 때문입니다. 따라서 보안 부팅을 해제함으로써 MOK을 해제하는 것이 가장 주된 관건이라고 볼 수 있습니다! 반드시 이 점을 유의하세요.
