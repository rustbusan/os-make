# 초기 설정

## 필수 요구사항

운영체제 개발을 시작하기 전에 다음 도구들이 설치되어 있어야 합니다.

### 1. Rust 설치

Rust는 공식 설치 스크립트를 사용하여 설치할 수 있습니다:

```bash
# Windows (PowerShell)
Invoke-WebRequest -Uri https://win.rustup.rs/x86_64 -OutFile rustup-init.exe
.\rustup-init.exe

# Linux/macOS
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

설치 후 다음 명령어로 확인:

```bash
rustc --version
cargo --version
```

### 2. Rust 타겟 설치

운영체제 개발을 위해 `no_std` 환경을 위한 타겟을 설치해야 합니다:

```bash
# x86_64 타겟 (일반적인 PC)
rustup target add x86_64-unknown-none

# 또는 커스텀 타겟 사용
```

### 3. QEMU 설치

QEMU는 가상 머신을 에뮬레이션하는 도구입니다.

#### Windows

```powershell
# Chocolatey 사용
choco install qemu

# 또는 직접 다운로드
# https://www.qemu.org/download/#windows
```

#### Linux (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install qemu-system-x86
```

#### macOS

```bash
brew install qemu
```

### 4. 부트로더 설정

GRUB 또는 커스텀 부트로더를 사용할 수 있습니다. 
Rust 생태계에서는 `bootloader` 크레이트를 사용하는 것이 일반적입니다.

### 5. 디버거 (선택사항)

GDB를 사용하여 커널을 디버깅할 수 있습니다:

#### Windows

```powershell
# Chocolatey 사용
choco install mingw

# 또는 WSL 사용
```

#### Linux

```bash
sudo apt-get install gdb
```

#### macOS

```bash
brew install gdb
```

## 프로젝트 초기화

### 1. 새 프로젝트 생성

```bash
cargo new --name os-make os-make
cd os-make
```

### 2. Cargo.toml 설정

`Cargo.toml` 파일을 다음과 같이 설정합니다:

```toml
[package]
name = "os-make"
version = "0.1.0"
edition = "2021"

[dependencies]
# 운영체제 개발에 필요한 크레이트들
bootloader = "0.12"

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

### 3. 커스텀 타겟 설정

`x86_64-unknown-none.json` 파일을 생성합니다:

```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "os": "none",
  "executables": true,
  "linker-flavor": "ld.lld",
  "linker": "rust-lld",
  "panic-strategy": "abort",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

### 4. 기본 커널 코드 작성

`src/main.rs` 파일을 다음과 같이 작성합니다:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

## 빌드 및 실행

### 빌드

```bash
cargo build --target x86_64-unknown-none.json
```

### QEMU에서 실행

```bash
qemu-system-x86_64 \
    -drive format=raw,file=target/x86_64-unknown-none/debug/os-make \
    -serial stdio
```

또는 더 간단하게:

```bash
qemu-system-x86_64 -kernel target/x86_64-unknown-none/debug/os-make
```

## Makefile 설정 (선택사항)

빌드 및 실행을 자동화하기 위해 `Makefile`을 생성할 수 있습니다:

```makefile
.PHONY: build run clean

TARGET = x86_64-unknown-none
KERNEL = target/$(TARGET)/debug/os-make

build:
	cargo build --target $(TARGET).json

run: build
	qemu-system-x86_64 -kernel $(KERNEL)

clean:
	cargo clean
```

## 문제 해결

### 일반적인 문제

1. **링커 오류**: `rust-lld`가 설치되어 있는지 확인
2. **QEMU 실행 오류**: 바이너리 형식 확인 (ELF 형식 필요)
3. **부팅 실패**: 부트로더 설정 확인

### 유용한 명령어

```bash
# 타겟 확인
rustup target list

# 크레이트 문서 확인
cargo doc --open

# 릴리즈 빌드
cargo build --release --target x86_64-unknown-none.json
```

## 다음 단계

- [프로젝트 구조](guide/project-structure.md) - 프로젝트 구조 설계
- [부트로더](guide/bootloader.md) - 부트로더 설정 및 구현

