# 부트로더

부트로더는 운영체제 커널을 메모리로 로드하고 실행하는 프로그램입니다.

## 부팅 프로세스 개요

1. **BIOS/UEFI**: 하드웨어 초기화 후 부트로더 실행
2. **부트로더**: 커널을 메모리로 로드
3. **커널**: 시스템 초기화 및 실행

## 부트로더 선택

### 옵션 1: GRUB (Multiboot)

GRUB는 널리 사용되는 부트로더입니다. Multiboot 표준을 따릅니다.

**장점**:
- 널리 사용되고 검증됨
- 다양한 파일 시스템 지원
- 설정이 비교적 간단

**단점**:
- 커널 크기 제한
- Multiboot 헤더 필요

### 옵션 2: bootloader 크레이트

Rust 생태계의 `bootloader` 크레이트를 사용할 수 있습니다.

**장점**:
- Rust로 작성됨
- 통합이 쉬움
- 최신 기능 지원

**단점**:
- 상대적으로 새로운 프로젝트

### 옵션 3: 커스텀 부트로더

직접 부트로더를 작성할 수 있습니다.

**장점**:
- 완전한 제어
- 학습 경험

**단점**:
- 복잡하고 시간 소모적
- 디버깅 어려움

## bootloader 크레이트 사용

### 1. 의존성 추가

`Cargo.toml`에 추가:

```toml
[dependencies]
bootloader = "0.12"
```

### 2. 부트 이미지 생성

`bootimage` 도구를 사용하여 부팅 가능한 이미지를 생성합니다:

```bash
cargo install bootimage
rustup component add llvm-tools-preview
```

### 3. 빌드 및 실행

```bash
# 빌드
cargo bootimage

# 실행
qemu-system-x86_64 -drive format=raw,file=target/x86_64-unknown-none/debug/bootimage-os-make.bin
```

## Multiboot 헤더 (GRUB 사용 시)

Multiboot 표준을 따르려면 커널에 Multiboot 헤더를 추가해야 합니다:

```rust
#[repr(align(4))]
#[repr(C)]
struct MultibootHeader {
    magic: u32,
    flags: u32,
    checksum: u32,
}

#[no_mangle]
#[link_section = ".multiboot"]
static MULTIBOOT_HEADER: MultibootHeader = MultibootHeader {
    magic: 0x1BADB002,
    flags: 0x00000003,
    checksum: -(0x1BADB002 + 0x00000003),
};
```

## 부트 정보 구조체

부트로더는 커널에 부팅 정보를 전달합니다:

```rust
#[repr(C)]
pub struct BootInfo {
    pub memory_map: MemoryMap,
    pub bootloader_name: &'static str,
    // 기타 부팅 정보
}
```

## 링커 스크립트

커널의 메모리 레이아웃을 정의하기 위해 링커 스크립트가 필요할 수 있습니다:

```ld
ENTRY(_start)

SECTIONS
{
    . = 1M;

    .boot :
    {
        *(.multiboot)
    }

    .text :
    {
        *(.text)
    }

    .rodata :
    {
        *(.rodata)
    }

    .data :
    {
        *(.data)
    }

    .bss :
    {
        *(.bss)
    }
}
```

## 커널 진입점

부트로더는 커널의 `_start` 함수를 호출합니다:

```rust
#[no_mangle]
pub extern "C" fn _start(boot_info: &'static BootInfo) -> ! {
    // 초기화 코드
    init(boot_info);
    
    // 메인 루프
    loop {}
}
```

## 문제 해결

### 부팅 실패

- 부트로더 설정 확인
- 커널 바이너리 형식 확인 (ELF)
- 메모리 레이아웃 확인

### 디버깅

QEMU의 디버그 모드 사용:

```bash
qemu-system-x86_64 \
    -kernel kernel.bin \
    -s -S  # GDB 서버 모드
```

그리고 다른 터미널에서:

```bash
gdb kernel.bin
(gdb) target remote :1234
```

## 다음 단계

- [커널 초기화](kernel-init.md) - 커널 초기화 과정 및 하드웨어 설정

