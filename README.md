# Ubuntu 24.04 부팅 오류 복구 정리 (KERNEL PANIC)

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

## ✅ 복구 준비물

*  **UEFI 부팅 가능한 Ubuntu Live USB**
---

## Step 0. BIOS 진입 및 Live USB 부팅

① BIOS 진입
- PC 부팅 직후 아래 키 중 하나를 반복 입력
  - `Del`, `F2`, `F10`, `F12`, `ESC` 등 제조사에 따라 다름

② Boot Mode 설정 확인
- `Boot Mode`: **UEFI**
- `CSM`: 비활성화 (가능하면)
- `Secure Boot`: 비활성화 권장

③ USB 부팅 우선순위 설정
- USB 디바이스를 부팅 순서 최상단으로 설정
- 예: `UEFI: <USB 이름>`

---
## Step 1. Ubuntu Live USB 부팅 → Try Ubuntu 선택

1. 부팅 후 "Ubuntu 설치" 화면 진입
2. **Try or Install Ubuntu** → 선택
   - `Try Ubuntu without installing` 없어도 괜찮음
3. "Next" 버튼 계속 누르다가 **Try Ubuntu** 선택
3. Ubuntu GUI 진입 후 Ctrl + Alt + T → 터미널 실행

---
## Step 2. 디스크 및 파티션 정보 확인

```bash
lsblk
sudo blkid
```
`lsblk` → 파티션 구조, 마운트 여부 확인
`sudo blkid` → UUID, 파일시스템 타입, 파티션 유형 확인

| 디바이스             | 설명      | 파일시스템 | 역할        |
| ---------------- | ------- | ----- | --------- |
| `/dev/nvme0n1`   | 전체 디스크  | -     | -         |
| `/dev/nvme0n1p1` | EFI 파티션 | vfat  | 부트 EFI    |
| `/dev/nvme0n1p2` | 루트 파티션  | ext4  | Ubuntu 루트 |


---

## Step 3. chroot 환경 준비
```bash
sudo mount /dev/nvme0n1p2 /mnt
sudo mount /dev/nvme0n1p1 /mnt/boot/efi
for d in /dev /proc /sys /run; do sudo mount --bind $d /mnt$d; done
sudo chroot /mnt
```

→ 터미널에  root@ubuntu:/# 로 나오면 정상

---

## Step 4. 복구 과정 
① NVMe 드라이버 포함 설정

```bash
echo nvme >> /etc/initramfs-tools/modules
```

② initramfs 재생성

```bash
update-initramfs -c -k 6.14.0-24-generic
```

> `-c`: 새로 생성, `-u`: 기존 업데이트
> `6.14.0-24-generic`은 현재 설치된 커널 버전



③ GRUB 설정 재적용

```bash
update-grub
```



④ GRUB 부트로더 재설치 (UEFI 대상)

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

## Step 5. chroot 종료 및 시스템 재부팅

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
