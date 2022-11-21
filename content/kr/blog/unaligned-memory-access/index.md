---
date: 2022-09-06
title: 정렬되지 않은 메모리 접근
description: 정렬되지 않은 메모리 접근Unaligned Memory Access 에 대해 알아봅니다
author: 권경환 ([@k](https://k.mononn.com))
tags: ["unaligned memory access"]
---

## Introduction
[Zephyr](https://github.com/zephyrproject-rtos/zephyr) 코드를 살펴보던 중 [다음
코드](https://git.o-g.at/tuxcoder/zephyr/commit/9ef64a31123e9f26a14dd589ebf1134eccbc67e9)를
만났습니다:

```c
#define UNALIGNED_GET(p)                  \
__extension__ ({                          \
    struct  __attribute__((__packed__)) { \
        __typeof__(*(p)) __v;             \
    } *__p = (__typeof__(__p)) (p);       \
    __p->__v;                             \
})
```

위 코드를 분해하면서, 메모리 주소가 접근 크기로 정렬되어 있지 않은 경우 어떻게
처리해야 하는지 알아보겠습니다.

## Unaligned Memory access 란
Instruction Set Architecture (ISA) 는 프로세서가 이해할 수 있는 명령어 집합을
정의하고 있습니다. 그중에는 바이트 단위로 메모리에 접근할 수 있는 명령어, 워드
단위로 접근할 수 있는 명령어, 하프워드halfword 단위로 접근할 수 있는 명령어
등이 있습니다. 메모리 접근에도 여러 명령이 있는 것처럼 메모리 접근은 여러
단위의 크기로 일어날 수 있습니다.

| 메모리 번지 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| ----------- |-- |-- |-- |-- | --| --| --| --| --| --|

위와 같은 메모리에 0번지를 워드 단위로 접근하면, 프로세서는 0 ~ 3 번지의
데이터를 읽어오게 됩니다. 하프워드의 경우 0 ~ 1 번지, 바이트일 경우 0 번지 만을
읽게 됩니다.

{{% alert title="Note" %}}
워드의 크기는 통상 시스템 버스 크기와 일치합니다. 16 비트 시스템에서 워드의
크기는 16 비트이고, 32 비트 시스템에서는 32 비트, 64 비트 시스템에서는 64
비트입니다. 여기에서는 32 비트 시스템을 가정합니다.
{{% /alert %}}

{{% alert title="Info" color="info" %}}
시스템 버스 크기와 동일한 메모리 크기 단위로 메모리 접근을 수행할 때 프로세서
클럭 사이클당 데이터 처리율이 가장 높습니다.
{{% /alert %}}

메모리는 0 번지부터 시작하고, 위 예시처럼 메모리 번지가 데이터 타입 크기 단위로
딱 떨어질 때 _정렬되어 있다_ 라고 합니다. 만약 1 번지를 워드, 하프워드, 바이트
단위로 읽게 된다면 바이트 단위 접근만 정렬되어 있는 겁니다($1\bmod1=0$). 2
번지를 읽게 된다면, 하프워드와 바이트 단위 접근이 정렬되어 있는
거고요($2\bmod1=0$, $2\bmod2=0$). 3 번지는 바이트 단위만, 4 번지는 워드,
하프워드, 바이트 모두에서 정렬되어 있습니다.

즉, $주소\bmod 타입크기 = 0$ 이면 정렬되어 있다고 볼 수 있습니다.

따라서 위 메모리 번지로의 접근은 아래와 같이 정리됩니다:

|   | 바이트  | 하프워드  | 워드      |
| --| ------- | --------- | --------- |
| 0 | aligned | aligned   | aligned   |
| 1 | aligned | unaligned | unaligned |
| 2 | aligned | aligned   | unaligned |
| 3 | aligned | unaligned | unaligned |
| 4 | aligned | aligned   | aligned   |
| 5 | aligned | unaligned | unaligned |
| 6 | aligned | aligned   | unaligned |
| 7 | aligned | unaligned | unaligned |
| 8 | aligned | aligned   | aligned   |
| 9 | aligned | unaligned | unaligned |

시스템에 따라 정렬되지 않은 메모리 접근은 허용되거나 fault 를 유발합니다. Fault
를 발생시키지 않더라도 정렬된 메모리 접근이 항상 같거나 더 나은 성능을
제공합니다.

## 왜 문제가 되나
그래서 정렬되지 않은 메모리 접근이 왜 문제가 될까요?

### C 표준
[C17 표준](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf) 66쪽의
6.2.8.1 과 74쪽의 6.3.2.3.7 을 읽어보면:

> Complete object types have alignment requirements which place restrictions on
> the addresses at which objects of that type may be allocated. An alignment is
> an implementation-defined integer value representing the number of bytes
> between successive addresses at which a given object can be allocated. An
> object type imposes an alignment requirement on every object of that type:
> stricter alignment can be requested using the _Alignas keyword.
>
> 완전한 오브젝트 타입은 해당 타입의 오브젝트가 할당될 수 있는 주소에 제한을
> 두는 정렬 요구 사항이 있습니다. 정렬은 주어진 오브젝트가 할당될 수 있는
> 연속적인 주소 사이의 바이트 수를 나타내는 정수 값입니다. 오브젝트 타입은 해당
> 타입의 모든 오브젝트에 정렬 요구 사항을 부과합니다. _Alignas 키워드를
> 사용하여 더 엄격한 정렬을 요청할 수 있습니다.

> A pointer to an object type may be converted to a pointer to a different object
> type. If the resulting pointer is not correctly aligned for the referenced
> type, the behavior is undefined. Otherwise, when converted back again, the
> result shall compare equal to the original pointer. When a pointer to an object
> is converted to a pointer to a character type, the result points to the lowest
> addressed byte of the object. Successive increments of the result, up to the
> size of the object, yield pointers to the remaining bytes of the object.
>
> 오브젝트 타입에 대한 포인터는 다른 오브젝트 타입에 대한 포인터로 변환될 수
> 있습니다. 참조된 타입에 대해 올바르게 정렬되지 않은 포인터 접근은 동작이
> 정의되어 있지 않습니다. [...] Otherwise 로 시작하는 문장은 이해가 잘 가지
> 않네요.

C 표준에서는 complete object type 에 대한 정렬 요구사항을 명시하고 있고, 참조
데이터 타입과 이를 가리키는 포인터가 올바르게 정렬되어 있지 않은 경우 정상적인
동작을 보장하지 않습니다.

### 프로세서 아키텍처
위에서 잠시 언급했듯이 정렬되지 않은 메모리 접근을 허용하지 않는 프로세서가
있습니다. ARMv6-M 의 경우 정렬되지 않은 메모리 접근에 대해 usage fault 를
발생시킵니다.

## 어떻게 회피하나
Unaligned memory access 를 지원하지 않는 프로세스에서 정렬되지 않은 메모리
주소로 multi-byte 접근을 할 경우 fault 가 발생합니다.

가령, `uint32_t val = *((uint32_t *)0x1)` 은 ARM 아키텍처의 경우 `ldr` 명령어로
변환되어 정렬되지 않은 메모리 접근을 발생시킵니다.

이를 우회하기 위해서는 컴파일러에서 해당 구문을 바이트 단위 명령어로 변환하도록
유도해야 하는데, 이때 `PACK_STRUCT` 또는 `memcpy` 를 사용합니다.

`memcpy` 는 C 표준에서 제공하는 방법이고, `pack` 속성은 컴파일러에게 바이트
단위 메모리 접근을 유도합니다.

언급된 서로 다른 세가지 방식의 메모리 접근은 실제로 다음 어셈블리 코드로
번역됩니다:

### 포인터 접근

```asm
# *(uint32_t *)ptr;
ldr     r0, [r0]
```

### memcpy

```asm
# memcpy(&dst, ptr, 4);
add     r0, sp, r2
bl      memcpy
```

### PACK_STRUCT

```asm
# __attribute__((__packed__))
# typedef struct { uint32_t b; } unaligned32_t;
# ((unaligned32_t *)ptr)->b;
ldrb    r3, [r0]
ldrb    r1, [r0, #1]
ldrb    r2, [r0, #2]
orr     r3, r3, r1, lsl #8
ldrb    r0, [r0, #3]
orr     r3, r3, r2, lsl #16
orr     r0, r3, r0, lsl #24
```

Zephyr 에서 사용한 `UNALIGNED_GET` 매크로는 가독성과 생산성을 높이기 위해
작성되었는데, 다소 복잡해보이는 이유는

1. `pack` 속성을 주입하기 위해 구조체를 사용
2. double evaluation 을 피하기 위해 새로운 변수에 파라미터를 할당해야하기 때문입니다

컴파일러 전처리기에서 발생하는 [double
evaluation](https://stackoverflow.com/questions/39439181/what-is-double-evaluation-and-why-should-it-be-avoided)
에 대해서는 링크를 참조해주세요.

## Conclusion
정렬되지 않은 메모리 접근을 지원하지 않는 프로세서에서 정렬되지 않은 메모리
주소로 multi-byte 접근이 필요한 경우, 컴파일러가 이를 바이트 단위 접근 명령으로
변환할 수 있도록 `PACK_STRUCT` 또는 `memcpy` 를 사용해야 합니다.

응용 개발의 경우 정렬되지 않은 메모리 접근 문제에 대해 고민할 필요는 많이
없습니다. 하지만, 펌웨어를 작성하거나 커널, 디바이스 드라이버 또는 메모리를
직접 제어해야 하는 등의 low-level 개발이 필요한 경우 고려해야합니다.

## References
- [C17 표준](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)
- [Unaligned accesses in C/C++: what, why and solutions to do it properly](https://blog.quarkslab.com/unaligned-accesses-in-cc-what-why-and-solutions-to-do-it-properly.html)
- [Take advantage of ARM unaligned memory access while writing clean C code](https://stackoverflow.com/questions/32062894/take-advantage-of-arm-unaligned-memory-access-while-writing-clean-c-code)
