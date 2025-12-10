# 프로젝트 구조

운영체제 프로젝트의 권장 디렉토리 구조와 각 모듈의 역할을 설명합니다.

## 디렉토리 구조

```
os-make/
├── Cargo.toml                 # 프로젝트 설정 및 의존성
├── Cargo.lock                 # 의존성 버전 고정
├── x86_64-unknown-none.json   # 커스텀 타겟 설정
├── Makefile                   # 빌드 자동화 (선택사항)
├── .cargo/
│   └── config.toml           # Cargo 설정
├── src/
│   ├── main.rs               # 커널 진입점
│   ├── lib.rs                # 라이브러리 루트 (선택사항)
│   ├── memory/
│   │   ├── mod.rs            # 메모리 관리 모듈
│   │   ├── paging.rs         # 페이징 구현
│   │   ├── heap.rs           # 힙 할당자
│   │   └── frame.rs          # 프레임 할당자
│   ├── interrupts/
│   │   ├── mod.rs            # 인터럽트 모듈
│   │   ├── idt.rs            # IDT 설정
│   │   ├── handlers.rs       # 인터럽트 핸들러
│   │   └── exceptions.rs     # 예외 처리
│   ├── process/
│   │   ├── mod.rs            # 프로세스 관리 모듈
│   │   ├── task.rs           # 태스크 구조체
│   │   ├── scheduler.rs      # 스케줄러
│   │   └── syscall.rs        # 시스템 콜
│   ├── fs/
│   │   ├── mod.rs            # 파일 시스템 모듈
│   │   ├── vfs.rs            # 가상 파일 시스템
│   │   └── ramfs.rs          # RAM 파일 시스템
│   ├── drivers/
│   │   ├── mod.rs            # 드라이버 모듈
│   │   ├── keyboard.rs       # 키보드 드라이버
│   │   ├── timer.rs          # 타이머 드라이버
│   │   └── vga.rs            # VGA 텍스트 모드
│   └── util/
│       ├── mod.rs            # 유틸리티 모듈
│       └── macros.rs         # 매크로 정의
├── bootloader/               # 부트로더 관련 파일
│   └── (부트로더 설정 파일)
├── tests/                    # 통합 테스트
│   └── integration_test.rs
└── docs/                     # 문서
    └── ...
```

## 주요 모듈 설명

### src/main.rs

커널의 진입점입니다. 부트로더가 이 함수를 호출합니다.

```rust
#![no_std]
#![no_main]

#[no_mangle]
pub extern "C" fn _start() -> ! {
    // 초기화 코드
    loop {}
}
```

### memory/

메모리 관리 관련 모듈:

- **paging.rs**: 페이지 테이블 설정 및 관리
- **heap.rs**: 동적 메모리 할당 (힙)
- **frame.rs**: 물리 메모리 프레임 할당

### interrupts/

인터럽트 처리 모듈:

- **idt.rs**: Interrupt Descriptor Table 설정
- **handlers.rs**: 인터럽트 핸들러 구현
- **exceptions.rs**: 예외 처리 (페이지 폴트, GPF 등)

### process/

프로세스 및 태스크 관리:

- **task.rs**: 태스크 구조체 및 상태 관리
- **scheduler.rs**: 프로세스 스케줄링 알고리즘
- **syscall.rs**: 시스템 콜 인터페이스

### fs/

파일 시스템 구현:

- **vfs.rs**: 가상 파일 시스템 인터페이스
- **ramfs.rs**: RAM 기반 파일 시스템 구현

### drivers/

하드웨어 장치 드라이버:

- **keyboard.rs**: PS/2 키보드 드라이버
- **timer.rs**: PIT (Programmable Interval Timer) 드라이버
- **vga.rs**: VGA 텍스트 모드 출력

## 모듈 간 의존성

```
main.rs
  ├── memory/        (메모리 관리)
  ├── interrupts/    (인터럽트 처리)
  ├── process/       (프로세스 관리)
  │   └── memory/    (메모리 할당 필요)
  ├── fs/            (파일 시스템)
  │   └── memory/    (버퍼 관리)
  └── drivers/       (장치 드라이버)
      └── interrupts/ (인터럽트 등록)
```

## Cargo.toml 예시

```toml
[package]
name = "os-make"
version = "0.1.0"
edition = "2021"

[dependencies]
bootloader = "0.12"
x86_64 = "0.14"
spin = "0.9"
volatile = "0.4"

[profile.dev]
panic = "abort"
opt-level = 0

[profile.release]
panic = "abort"
opt-level = 3
lto = true
```

## 빌드 설정

`.cargo/config.toml` 파일에서 빌드 설정을 관리합니다:

```toml
[build]
target = "x86_64-unknown-none.json"

[target.x86_64-unknown-none]
runner = "qemu-system-x86_64 -kernel"
```

## 테스트 구조

통합 테스트는 `tests/` 디렉토리에 위치합니다:

```rust
// tests/integration_test.rs
#![no_std]
#![no_main]

#[test_case]
fn test_basic() {
    // 테스트 코드
}
```

## 다음 단계

- [부트로더](bootloader.md) - 부트로더 설정 및 구현
- [커널 초기화](kernel-init.md) - 커널 초기화 과정

