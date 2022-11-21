---
date: 2022-11-25
title: 인라인 어셈블리
description: C/C++ 소스 코드 내에서 어셈블리를 인라인으로 사용하는 방법을 설명합니다.
author: 권경환 ([@k](https://k.mononn.com))
tags: ["assembly", "compiler"]
---

C, C++ 그리고 Rust 와 같은 언어에서는 소스 코드 내에서 어셈블리 명령어를 사용할
수 있는 인라인 어셈블리를 지원합니다.

인라인 어셈블리는 다음과 같은 경우에 유용할 수 있습니다:
- 인터럽트 활성/비활성화와 같은 특수한 하드웨어 기능을 실행해야 할 때
- 컴파일러에서 유발하는 오버헤드를 피하고자 할 때
- 컴파일러에서 생성하는 함수 프롤로그나 에필로그 생성을 피하고 싶을 때
- 최적화 방지를 위한 메모리 배리어를 삽입하고자 할 때

{{% alert title="Info" color="info" %}}
적당한 한글 표현을 찾지 못해 constraint 와 modifier, 그리고 clobber 는 영문
그대로 표기합니다. constraint 는 인자의 타입으로, modifier 는 인자의 속성으로
이해하면 될 것 같습니다. clobber 는 부수효과side effect로 영향받는 레지스터를
나타냅니다.

modifier 도 constraint 일종이지만, 편의상 분리해서 표기했습니다.
{{% /alert %}}

## 문법syntax

인라인 어셈블리는 다음과 같은 형식으로 이루어집니다:

```asm
__asm__ (어셈블리 코드
    : 출력인자output operands             (optional)
    : 입력인자input operands              (optional)
    : clobbered registers list          (optional)
    );
```

`__asm__` 대신 `asm` 을 사용한 코드도 볼 수 있는데요, 호환성을 고려한다면
`__asm__` 사용을 추천합니다. `asm` 은 컴파일러 확장 기능인 반면, `__asm__` 은
예약어reserved identifier이기 때문입니다. [GCC
매뉴얼](https://gcc.gnu.org/onlinedocs/gcc/Alternate-Keywords.html#Alternate-Keywords)과
[C 표준](https://web.archive.org/web/20181230041359if_/http://www.open-std.org/jtc1/sc22/wg14/www/abq/c17_updated_proposed_fdis.pdf)
7.1.3 Reserved identifiers 에서 관련 내용을 찾을 수 있습니다.

_optional_ 로 표기된 항목은 생략할 수 있습니다만, 아래 clobber 항목처럼
뒤따라오는 항목이 있는 경우 구분자인 콜론`:`을 반드시 유지해야 합니다:

```asm
asm("":::"memory");
```

편의상 아래 [코드
예제](https://github.com/onkwon/yaos/blob/af7f659959c68ae3144f7f696aa2a88b92e70c01/arch/armv7-m/include/atomic.h#L13)를
기준으로 설명합니다(ARM 아키텍처에서 atomic 연산을 위해 제공하는 어셈블리
명령어를 사용하는 예제입니다):

```asm
__asm__ __volatile__("ldrex %0, [%1]"
	: "=r"(result)
	: "r"(addr)
	: "cc", "memory"
);
```

{{% alert title="Note" %}}
예제는 ARM 아키텍처 기반 GCC 기준으로 작성되었습니다.
{{% /alert %}}

### 어셈블리 코드
예제에서 `ldrex %0, [%1]` 부분으로, 어셈블리 명령을 포함하는 문자열
리터럴입니다. 

문자열 리터럴에 표현된 `%숫자` 는 나열된 입/출력인자의 순서입니다. 0 부터
시작하기 때문에 예제에서는 `%0` 이 `result` 를 가리키고, `%1` 이 `addr` 을
가리킵니다.

문자열 리터럴은
[아래](https://github.com/onkwon/yaos/blob/c9571f22b310002e16bb012215dfbfb84a1e264b/arch/armv7-m/include/syscall.h#L4)와
같이 하나의 어셈블리 명령뿐만 아니라 여러 개의 어셈블리 명령을 포함할 수
있습니다:

```asm
__asm__ __volatile__(
	"svc %0  \n\t"
	"bx  lr  \n\t"
	:: "I"(n) : "memory"
);
```

각 명령은 문자열로 어셈블러에 전달되기 때문에 어셈블러 포맷에 맞게 문장 끝에
`\n\t` 을 추가했습니다. GCC 에서 세미콜론은 명령 구분자로 사용되기 때문에
세미콜론을 사용할 수도 있습니다.

위 문법syntax 템플릿에는 없던 `__volatile__` qualifier 가 예제에서
사용됐는데요. 이는 컴파일러 최적화로 구문이 제거되지 않도록 하기 위해
사용되었습니다. 컴파일러는 인라인 어셈블리 구문을 해석할 수 없기 때문에
*출력인자*가 지정되지 않으면 해당 구문을 no-op 으로 이해하고 최적화 과정에서 구문
전체를 제거할 수 있기 때문입니다.

### 출력인자ouput operands
예제에서 `: "=r"(result)` 부분으로, 어셈블리 코드 결과가 저장되는 변수입니다.

`: "=r"(result)` 에서 `=` 는 modifier, 바로 뒤에 붙어있는 `r` 는 constraint,
그리고 `result` 는 C/C++ 변수명입니다.

예제를 풀이하자면, 명령의 결과를 general register 중 하나에 저장하고, 변수
`result` 로 접근할 수 있게한다는 겁니다.

풀이를 이해하기 위해서는 modifier 와 constraint 를 알아야 하는데요, 얘네는
입력인자에도 공통으로 쓰이는 내용이므로 입력인자 설명 [뒤쪽으로
배치](#operand-modifier)했습니다. constraint 는 아키텍처마다 달라서 리스트가
길기도 하고요.

### 입력인자input operands
예제에서 `: "r"(addr)` 부분으로, `addr` 변수의 값이 읽기전용으로 어셈블리
코드에서 사용된다는 걸 말합니다.

예제에서 `addr` 은 컴파일러에 의해 general register 중 하나로 어셈블리 코드에서
사용할 수 있게 되는데요, 컴파일러는 clobber 리스트에 지정되어 있지 않은
레지스터를 할당합니다.

{{% alert title="Warning" color="warning" %}}
어셈블리 구문에서 입력전용 인자를 임의로 변경할 경우 컴파일러는 이를 인지할 수
없다는 점을 명심해야 합니다. 입력전용 인자의 경우 컴파일러는 어셈블리 구문 전과
후의 값이 동일하다고 가정합니다.
{{% /alert %}}

### Clobbered Register List
컴파일러는 어셈블리 구문에 전달된 출력인자와 입력인자를 위해 적당한 레지스터를
선택해 어셈블리 코드에서 사용할 수 있게 합니다. 그리고 컴파일러는 출력인자
항목만이 어셈블리 구문에서 변경되는 유일한 인자라고 파악합니다.

반면, 어셈블리 코드에서는 어떤 부수효과side effect도 발생할 수 있습니다. 예컨대
두 값을 스왑하기위해 임시 레지스터를 사용하는 것처럼요. 이런 식으로 어셈블리
코드에서 컴파일러에 알려지지 않은 레지스터를 사용한 경우 curruption 이 발생할
수 있습니다. 이를 방지하기 위해 _clobbered register list_ 가 존재합니다. 여기에
어떤 레지스터들이 어셈블리 코드에서 사용됐는지 적어두면 컴파일러는 어셈블리
구문을 실행하기 전에 해당 레지스터들을 백업하고, 구문이 끝난 뒤 복구하는 작업을
합니다.

예제에서 사용한 `"cc"` 와 `"memory"` 는 특수 용도입니다:

- `cc`
  - 어셈블리 구문의 결과로 상태 레지스터 변경이 발생함을 알립니다.
- `memory`
  - 어셈블리 구문에서 입력/출력인자에 리스트되지 않은 메모리 접근이 발생함을
    알립니다. 이는 컴파일러 메모리 배리어 역할을 수행합니다.

{{% alert title="Info" color="info" %}}
컴파일러 최적화와 관련해 `volatile` 과 `memory` 두 가지가 언급되었는데요.
`volatile` 은 최적화에서 코드가 제거되는 걸 방지해주고, `memory` 는 최적화에서
구문 위치 변경을 방지해줍니다.
{{% /alert %}}

## Operand Modifier
다음과 같은 modifier 가 있습니다. 인자에 modifier 가 따로 전달되지 않은 경우
인자는 읽기전용으로 해석됩니다:

- `=`
  - 출력전용 인자operand를 뜻합니다. 어셈블리 구문에서 출력이 지정되기 전까지
    어떤 유효한 값도 가지지 않습니다.
- `+`
  - 출력과 입력 모두에 사용되는 인자를 뜻합니다.
- `&`
  - 출력인자에만 사용할 수 있는 _earlyclobber_ 로, 컴파일러가 출력인자와
    입력인자에 서로 다른 레지스터를 할당하도록 유도합니다. 컴파일러는
    기본적으로 입력인자를 사용한 뒤, 그 결과를 출력인자에 저장한다고 가정하기
    때문에 필요한 modifier 인데요. 단일 어셈블리 명령어가 아닌 여러 개의
    명령어로 구성된 어셈블리 구문일 경우, 입력인자를 사용하기 전에 출력인자를
    먼저 사용할 수도 있다고 알리는 역할을 합니다. 그렇지 않은 경우 출력인자와
    입력인자에 동일한 레지스터를 할당할 수 있습니다.

## Constraints
_modifier_ 는 _constraint_ 앞에 표기됩니다. _modifier_ 가 없는 인자는 읽기전용
인자로 처리합니다.

### 공통simple constraints

{{% alert title="Note" %}}
전체 리스트를 포함하지 않습니다. 전체 리스트는 [참고자료](#참고자료)를 참고하세요.
{{% /alert %}}

| constraint | 설명                   |
| ---------- | ---------------------- |
| `m`        | 메모리 인자            |
| `r`        | 일반 레지스터 인자     |
| `i`        | 상수(심볼릭 상수 포함) |
| `n`        | 지원범위내의 상수로써 `i` 보다 `n` 사용. 워드 사이즈보다 작은 상수를 지원하기 때문 |
| `F`        | 부동소수점 상수        |
| `g`        | special 레지스터를 제외한 일반 레지스터 |

### 아키텍처별machine constraints
#### ARM

| Constraint | 설명                               |
| ---------- | ---------------------------------- |
| `f`        | 부동소수점 레지스터                |
| `G`        | 부동소수점 상수                    |
| `h`        | `r8..r15` 레지스터                 |
| `I`        | 0 ~ 255 사이의 상수                |
| `J`        | -4095 ~ 4095 사이의 상수           |
| `K`        | `I` 에 만족하는 값의 1 의 보수     |
| `L`        | `I` 에 만족하는 값의 2 의 보수     |
| `l`        | `r0..r7` 레지스터                  |
| `w`        | `s0..s31` 벡터 부동소수점 레지스터 |

#### AVR

{{% alert title="Note" %}}
x 레지스터는 r27:r26, y 레지스터는 r29:r28, z 레지스터는 r31:30 입니다.
{{% /alert %}}

| Constraint | 설명               |
| ---------- | ------------------ |
| `a`        | r16 ~ r23          |
| `b`        | y, z               |
| `d`        | r16 ~ r31          |
| `e`        | x, y, z            |
| `q`        | SPH:SPL            |
| `r`        | r0 ~ r31           |
| `t`        | r0                 |
| `w`        | r24, r26, r28, r30 |
| `x`        | x                  |
| `y`        | y                  |
| `z`        | z                  |
| `G`        | 0.0                |
| `I`        | 0 ~ 63             |
| `J`        | -63 ~ 0            |
| `K`        | 2                  |
| `L`        | 0                  |
| `l`        | r0 ~ r15           |
| `M`        | 0 ~ 255            |

AVR 의 경우, 추가로 컴파일러에 미리 정의된 심볼이 있습니다.

| 심볼           | 레지스터                     |
| -------------- | ---------------------------- |
| `__SREG__`     | 0x3F 상태 레지스터           |
| `__SP_H__`     | 0x3E 스택 포인터 상위 바이트 |
| `__SP_L__`     | 0x3D 스택 포인터 하위 바이트 |
| `__tmp_reg__`  | r0                           |
| `__zero_reg__` | r1                           |

[참고자료](#참고자료)에서 AVR 니모닉별 관련 constraints 리스트를 찾아볼 수 있습니다.

## 추가내용
### Symbolic Name

어셈블리 코드에서 인자를 가리키기 위해 `%숫자` 를 사용하지 않고 심볼릭을 대신
지정해서 사용할 수 있습니다. 예제코드에서 숫자를 심볼릭으로 대체하면 아래와
같습니다:

```asm
__asm__ __volatile__("ldrex %[dst], [%[src]]"
	: [dst] "=r" (result)
	: [src] "r" (addr)
	: "cc", "memory"
);
```

## 참고자료
- https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html
- https://asm.sourceforge.net/articles/rmiyagi-inline-asm.txt
- https://www.nongnu.org/avr-libc/user-manual/inline_asm.html
