# 커널 초기화

커널이 부팅된 후 시스템을 초기화하는 과정을 설명합니다.

## 초기화 순서

커널 초기화는 다음 순서로 진행됩니다:

1. **기본 설정**: 스택 포인터, 세그먼트 레지스터 설정
2. **메모리 관리**: 페이지 테이블 설정, 힙 초기화
3. **인터럽트**: IDT 설정, 인터럽트 활성화
4. **장치 초기화**: 타이머, 키보드 등 하드웨어 초기화
5. **사용자 인터페이스**: 쉘 또는 GUI 초기화

## 기본 초기화 코드

```rust
#[no_mangle]
pub extern "C" fn _start(boot_info: &'static BootInfo) -> ! {
    // 1. 초기 스택 설정 (이미 부트로더가 설정했을 수 있음)
    
    // 2. 초기화 함수 호출
    init(boot_info);
    
    // 3. 메인 루프
    main_loop();
}

fn init(boot_info: &'static BootInfo) {
    // 메모리 관리자 초기화
    memory::init(boot_info);
    
    // 인터럽트 초기화
    interrupts::init();
    
    // 장치 드라이버 초기화
    drivers::init();
    
    // 파일 시스템 초기화
    fs::init();
    
    println!("커널 초기화 완료!");
}
```

## 메모리 초기화

### 페이지 테이블 설정

```rust
use x86_64::structures::paging::PageTable;

pub fn init_memory(boot_info: &BootInfo) {
    // 부트로더가 제공한 메모리 맵 사용
    let memory_map = &boot_info.memory_map;
    
    // 페이지 테이블 생성
    let mut mapper = unsafe { create_page_table() };
    
    // 힙 영역 매핑
    map_heap(&mut mapper);
    
    // 힙 할당자 초기화
    init_heap();
}
```

### 힙 초기화

```rust
use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();

pub fn init_heap() {
    const HEAP_START: usize = 0x_4444_4444_0000;
    const HEAP_SIZE: usize = 100 * 1024; // 100 KB
    
    unsafe {
        ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE);
    }
}
```

## 인터럽트 초기화

### IDT 설정

```rust
use x86_64::structures::idt::InterruptDescriptorTable;

static mut IDT: InterruptDescriptorTable = InterruptDescriptorTable::new();

pub fn init_idt() {
    unsafe {
        IDT.breakpoint.set_handler_fn(breakpoint_handler);
        IDT.double_fault.set_handler_fn(double_fault_handler);
        IDT.page_fault.set_handler_fn(page_fault_handler);
        IDT[0x20].set_handler_fn(timer_interrupt_handler);
        IDT[0x21].set_handler_fn(keyboard_interrupt_handler);
        
        IDT.load();
    }
}

pub fn enable_interrupts() {
    x86_64::instructions::interrupts::enable();
}
```

### 예외 핸들러

```rust
extern "x86-interrupt" fn breakpoint_handler(
    stack_frame: InterruptStackFrame
) {
    println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
}

extern "x86-interrupt" fn double_fault_handler(
    stack_frame: InterruptStackFrame,
    _error_code: u64,
) -> ! {
    panic!("EXCEPTION: DOUBLE FAULT\n{:#?}", stack_frame);
}

extern "x86-interrupt" fn page_fault_handler(
    stack_frame: InterruptStackFrame,
    error_code: PageFaultErrorCode,
) {
    use x86_64::registers::control::Cr2;
    
    println!("EXCEPTION: PAGE FAULT");
    println!("Accessed Address: {:?}", Cr2::read());
    println!("Error Code: {:?}", error_code);
    println!("{:#?}", stack_frame);
    panic!("Page fault occurred");
}
```

## 장치 초기화

### 타이머 초기화

```rust
pub fn init_timer() {
    // PIT (Programmable Interval Timer) 설정
    // 1ms마다 인터럽트 발생하도록 설정
    const TIMER_FREQUENCY: u32 = 1000; // Hz
    
    unsafe {
        let divisor = 1193180 / TIMER_FREQUENCY;
        let bytes = divisor.to_le_bytes();
        
        // PIT 제어 레지스터에 쓰기
        x86_64::instructions::port::Port::new(0x43).write(0x36u8);
        x86_64::instructions::port::Port::new(0x40).write(bytes[0]);
        x86_64::instructions::port::Port::new(0x40).write(bytes[1]);
    }
}
```

### 키보드 초기화

```rust
pub fn init_keyboard() {
    // PS/2 키보드 컨트롤러 초기화
    unsafe {
        // 키보드 활성화
        x86_64::instructions::port::Port::new(0x64).write(0xAEu8);
    }
}
```

### VGA 텍스트 모드 초기화

```rust
pub fn init_vga() {
    // VGA 텍스트 모드 버퍼는 0xB8000에 위치
    // 80x25 문자, 각 문자는 2바이트 (문자 + 속성)
    let vga_buffer = 0xB8000 as *mut u8;
    
    // 화면 클리어
    for i in 0..(80 * 25 * 2) {
        unsafe {
            *vga_buffer.add(i) = 0;
        }
    }
}
```

## 초기화 완료 후

초기화가 완료되면 메인 루프로 진입합니다:

```rust
fn main_loop() -> ! {
    println!("커널이 정상적으로 실행 중입니다.");
    println!("명령을 입력하세요:");
    
    loop {
        // 키보드 입력 대기
        if let Some(key) = keyboard::read_key() {
            handle_key(key);
        }
        
        // 스케줄러 실행 (프로세스 관리 구현 후)
        // scheduler::run();
        
        // CPU 절전 모드
        x86_64::instructions::hlt();
    }
}
```

## 초기화 체크리스트

초기화 과정에서 다음을 확인해야 합니다:

- [ ] 스택이 올바르게 설정되었는가?
- [ ] 페이지 테이블이 올바르게 설정되었는가?
- [ ] 힙 할당자가 작동하는가?
- [ ] 인터럽트가 활성화되었는가?
- [ ] 예외 핸들러가 등록되었는가?
- [ ] 타이머 인터럽트가 발생하는가?
- [ ] 키보드 입력이 처리되는가?
- [ ] 화면 출력이 정상인가?

## 디버깅 팁

### 초기화 단계별 출력

각 초기화 단계마다 출력을 추가하여 어느 단계에서 문제가 발생하는지 확인:

```rust
fn init(boot_info: &'static BootInfo) {
    println!("[1/5] 메모리 초기화 중...");
    memory::init(boot_info);
    println!("[1/5] 완료");
    
    println!("[2/5] 인터럽트 초기화 중...");
    interrupts::init();
    println!("[2/5] 완료");
    
    // ...
}
```

### QEMU 모니터 사용

QEMU의 모니터를 통해 시스템 상태 확인:

```bash
qemu-system-x86_64 -kernel kernel.bin -monitor stdio
```

모니터에서 사용 가능한 명령어:
- `info registers`: CPU 레지스터 상태
- `info mem`: 메모리 매핑 정보
- `info irq`: 인터럽트 상태

## 다음 단계

초기화가 완료되면 다음 기능을 구현할 수 있습니다:

- 프로세스 관리 및 멀티태스킹
- 파일 시스템 구현
- 추가 장치 드라이버 개발
- 사용자 인터페이스 구현

