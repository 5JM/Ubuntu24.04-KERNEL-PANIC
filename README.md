# Ubuntu 24.04 부팅 오류 복구 정리 KERNEL PANIC 복구 과정 기록


# ✅ Ubuntu 부팅 오류 복구 정리 (KERNEL PANIC)

> 오류 메시지:
> `KERNEL PANIC! VFS: Unable to mount root fs on unknown-block(0,0)`
>
> 주요 원인:
>
> * initramfs 누락 또는 손상
> * NVMe 드라이버 미포함
> * GRUB 부트로더 설정 불완전
> * EFI 부트 항목 누락

---

## 🧾 복구 준비물

* ✅ **UEFI 부팅 가능한 Ubuntu Live USB**
* ✅ 인터넷 연결 (가능 시)
* ✅ 루트 디스크/파티션 정보 확인용 명령어:

  * `lsblk` : 블록 디바이스 구조 확인
  * `blkid` : 파일시스템 종류 및 UUID 확인
* ✅ 대상 디스크 파티션 구성 예시:

| 디바이스             | 설명          | 파일시스템 | 역할          |
| ---------------- | ----------- | ----- | ----------- |
| `/dev/nvme0n1`   | NVMe 디스크 전체 | -     | 디스크 본체      |
| `/dev/nvme0n1p1` | 파티션 1       | vfat  | EFI (boot)  |
| `/dev/nvme0n1p2` | 파티션 2       | ext4  | Ubuntu root |

---

## 🔍 복구 전 파티션 정보 확인

```bash
lsblk
```

→ 파티션 구조, 마운트 여부 확인

```bash
sudo blkid
```

→ UUID, 파일시스템 타입, 파티션 유형 확인
→ `/dev/nvme0n1p1` 은 `TYPE="vfat"` (EFI), `/dev/nvme0n1p2` 는 `TYPE="ext4"` (루트)

---

## 🔧 복구 절차 요약

### ① UEFI 부팅 확인

```bash
[ -d /sys/firmware/efi ] && echo "UEFI mode" || echo "BIOS mode"
```

→ "UEFI mode" 가 출력되면 정상

---

### ② chroot 환경 준비

```bash
sudo mount /dev/nvme0n1p2 /mnt
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
for d in /dev /proc /sys /run; do sudo mount --bind $d /mnt$d; done
sudo chroot /mnt
```

→ 터미널에  root@ubuntu:/# 로 나오면 정상

---

### ③ NVMe 드라이버 포함 설정

```bash
echo nvme >> /etc/initramfs-tools/modules
```

---

### ④ initramfs 재생성

```bash
update-initramfs -c -k 6.14.0-24-generic
```

> `-c`: 새로 생성, `-u`: 기존 업데이트
> `6.14.0-24-generic`은 현재 설치된 커널 버전

---

### ⑤ GRUB 설정 재적용

```bash
update-grub
```

---

### ⑥ GRUB 부트로더 재설치 (UEFI 대상)

```bash
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

> ⚠️ 경고 메시지 발생 가능:
>
> ```
> EFI variables cannot be set on this system.
> You will have to complete the GRUB setup manually.
> ```
>
> → 무시 가능. 이미 EFI 항목이 등록되어 있었던 것으로 판단됨

---

### ⑦ chroot 종료 및 시스템 재부팅

```bash
exit
sudo umount -R /mnt
sudo reboot
```

---

## 📦 전체 조치 요약

| 조치 항목                        | 적용 여부     | 설명                        |
| ---------------------------- | --------- | ------------------------- |
| UEFI 모드 확인                   | ✅         | `/sys/firmware/efi` 확인    |
| 파티션 정보 확인 (`lsblk`, `blkid`) | ✅         | 루트 및 EFI 파티션 위치 확인        |
| EFI 파티션 마운트                  | ✅         | `/mnt/boot/efi`           |
| NVMe 드라이버 추가                 | ✅         | `initramfs-tools/modules` |
| initramfs 재생성                | ✅         | `update-initramfs`        |
| GRUB 설정 갱신                   | ✅         | `update-grub`             |
| GRUB 재설치 (UEFI 대상)           | ✅ (경고 발생) | `grub-install`            |
| efibootmgr 수동 등록             | ❌ (시도 실패) | 하지만 자동 등록되어 문제 없음         |
| 재부팅 후 정상 부팅                  | ✅         | 문제 해결 완료                  |

---
