---
date: 2022-08-18
title: CBOR 바이너리 인코딩
description: 경량 바이너리 포맷 CBOR 에 대해 알아봅니다
author: 권경환 ([@k](https://k.mononn.com))
tags: ["serialization", "CBOR"]
---

## 들어가기
Concise Binary Object Representation(CBOR) 의 목적과 특징은 다음 세 가지로
요약할 수 있습니다.

1. 인터넷 표준으로 정의된 기본 데이터 타입과 자료구조를 바이너리 포맷으로
   명확하게 표현합니다
2. 제한된 메모리와 프로세서 자원을 가진 시스템에서도 원활히 동작하도록 인코더와
   디코더를 경량 하게 구현합니다
3. 데이터를 스키마schema description 없이 디코딩 합니다

문자열로 이루어진 JSON 의 가독성을 제외하면 CBOR 는 JSON 의 바이너리 버전이라고
볼 수 있습니다. JSON 이 지원하는 모든 타입과 데이터를 지원하고 있고요.

### 용례
AWS, GCP 와 같은 클라우드 IoT 서비스에서 통신 포맷으로 CBOR 를 지원하고
있습니다. IoT 디바이스의 메모리 제약과 네트워크 대역을 고려한 선택인 것
같습니다.

### 관련 포맷
비슷한 용도로 아래와 같은 포맷들이 있습니다:

- [MessagePack](https://msgpack.org/)
- [BSON](https://bsonspec.org/)

## JSON 과 바이너리 인코딩 비교
### JSON
바이너리 데이터 `[0x00,0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09]` 를 base64
로 인코딩한 JSON 페이로드는 아래와 같이 27 바이트입니다.

```json
{"data":"AAECAwQFBgcICQ=="}
```

### CBOR
동일한 데이터를 CBOR 로 인코딩하면 아래와 같이 14 바이트가 됩니다.

`A161788A00010203040506070809`

바이트별 해석은 아래와 같습니다:

```
A1       # map(1)
   61    # text(1)
      78 # "x"
   8A    # array(10)
      00 # unsigned(0)
      01 # unsigned(1)
      02 # unsigned(2)
      03 # unsigned(3)
      04 # unsigned(4)
      05 # unsigned(5)
      06 # unsigned(6)
      07 # unsigned(7)
      08 # unsigned(8)
      09 # unsigned(9)
```

문자열 기반 포맷인 JSON 에 비해 바이너리를 다루는 CBOR 가 메모리 사용량이 적을
수밖에 없고, 그에 따라 연산속도도 훨씬 빠릅니다.

## 사용예제
예제에서는 정수형 시간과 바이너리 문자열로 구성된 간단한 데이터를 전송합니다.
대부분의 IoT 디바이스들이 C 언어로 구현되기 때문에 예제에서는 C 언어를
사용합니다.

{{% alert title="Note" %}}
안정적인 네트워킹을 위해서는 프레이밍 포맷이 추가되어야 하지만, CBOR 사용방법에
집중하기 위해 관련 내용은 포함하지 않습니다.
{{% /alert %}}

### CBOR C 라이브러리
[C 로 구현된 여러
라이브러리](https://en.wikipedia.org/wiki/CBOR#Implementations)가 있습니다.
그중 [QCBOR](https://github.com/laurencelundblade/QCBOR) 와
[TinyCBOR](https://github.com/intel/tinycbor) 가 다른 라이브러리에 비해
상대적으로 많이 알려져 있습니다.

여기에서는 [libmcu CBOR](https://github.com/libmcu/cbor) 를 사용합니다. 메모리
파편화를 유발하는 동적할당을 사용하지 않는 한편, 메모리 사용량이 매우 낮고,
사용방법도 상대적으로 간단하기 때문입니다.

#### 프로젝트에 라이브러리 추가하기
다음 4가지 방법으로 라이브러리를 프로젝트에 추가할 수 있습니다:

1. 깃 서브모듈
2. 깃 클론
3. 소스 다운로드
4. CMake FetchContent

아래에서는 편의상 터미널 커맨드를 사용합니다만, modern IDE 는 똑같은 기능을
제공하는 GUI 인터페이스를 지원합니다.

##### 깃 서브모듈

```bash
$ git submodule add https://github.com/libmcu/cbor.git <THIRD_PARTY_DIR>
```

##### 깃 클론

```bash
$ git clone https://github.com/libmcu/cbor.git <THIRD_PARTY_DIR>
```

##### 소스 다운로드

```bash
$ wget https://github.com/libmcu/cbor/archive/refs/heads/main.zip
```

##### CMake FetchContent

```CMake
include(FetchContent)
FetchContent_Declare(cbor
                      GIT_REPOSITORY https://github.com/libmcu/cbor.git
                      GIT_TAG main
)
FetchContent_MakeAvailable(cbor)
target_compile_options(cbor PUBLIC <target-specific-options>)

...

target_link_libraries(your-target
	...
	cbor
)
```

사용하는 개발환경에 따라 Makefile, CMake 파일을 수정하거나, 디렉토리를
추가합니다.

##### Make

```make
CBOR_ROOT ?= <THIRD_PARTY_DIR>/cbor
include $(CBOR_ROOT)/cbor.mk

SRCS += $(CBOR_SRCS)
INCS += $(CBOR_INCS)
```

##### CMake

```CMake
set(CBOR_ROOT <THIRD_PARTY_DIR>/cbor)
include(${CBOR_ROOT}/cbor.cmake)

# 라이브러리로 추가할 경우

add_library(cbor OBJECT ${CBOR_SRCS})
target_include_directories(cbor PUBLIC {CBOR_INCS})

...

target_link_libraries(<YOUR-TARGET>
	...
	cbor
)

# 소스로 추가할 경우

add_executable(<YOUR-PROJECT> ... ${CBOR_SRCS})
target_include_directories(<YOUR-PROJECT> PRIVATE ... ${CBOR_INCS})
```

이제 컴파일 준비가 됐으니 간단한 예제를 작성해보겠습니다.

### 인코딩하기
송신 데이터는 다음과 같이 구성합니다:

```json
{
	"time": 1660530996,
	"data": [ 0x00, ... 0x09 ]
}
```

라이브러리에서 제공하는 다음 함수들을 사용해 데이터 타입별로 인코딩할 수
있습니다:

- 정수
	- `cbor_encode_unsigned_integer()`
	- `cbor_encode_negative_integer()`
- 부동소수점
	- `cbor_encode_float()`
	- `cbor_encode_double()`
- 문자열
	- `cbor_encode_text_string()`
- 바이트열
	- `cbor_encode_byte_string()`
- 배열
	- `cbor_encode_array()`
- 맵 혹은 딕셔너리
	- `cbor_encode_map()`
- 불린
	- `cbor_encode_bool()`
- 널
	- `cbor_encode_null()`
- undefined
	- `cbor_encode_undefined()`

위에서 미리 정의한 데이터 구조는 맵, 문자열, 정수를 포함하고 있기 때문에 위
함수들 중 아래 함수를 사용합니다:

- `cbor_encode_map()`
- `cbor_encode_text_string()`
- `cbor_encode_unsigned_integer()`

인코딩을 위한 Writer 와 버퍼를 먼저 초기화하고, 위 함수를 사용해 다음과 같이
데이터를 인코딩합니다.

```c
uint8_t buf[BUFSIZE];
cbor_writer_t writer;

cbor_writer_init(&writer, buf, sizeof(buf));

cbor_encode_map_indefinite(&writer);
  cbor_encode_text_string(&writer, "time");         /* 1st key */
  cbor_encode_unsigned_integer(&writer, unixtime);  /* 1st value */
  cbor_encode_text_string(&writer, "data");         /* 2nd key */
  cbor_encode_byte_string(&writer, data, data_len); /* 2nd value */
cbor_encode_break(&writer);

transport_send(cbor_writer_get_encoded(&writer), cbor_writer_len(&writer));
```

이렇게 인코딩한 페이로드는 약 27 바이트입니다. "약"이라고 한 이유는 데이터
타입과 실제값에 따라 인코딩 사이즈가 달라지기 때문입니다.

```
A2             # map(2)
   64          # text(4)
      74696D65 # "time"
   1A 62F9B134 # unsigned(1660530996)
   64          # text(4)
      64617461 # "data"
   8A          # array(10)
      00       # unsigned(0)
      01       # unsigned(1)
      02       # unsigned(2)
      03       # unsigned(3)
      04       # unsigned(4)
      05       # unsigned(5)
      06       # unsigned(6)
      07       # unsigned(7)
      08       # unsigned(8)
      09       # unsigned(9)
```

{{% alert title="Info" color="info" %}}
실제값이 작은 정수일 때 인코딩 사이즈에서 이득을 볼 수 있습니다. 4-byte
정수라도 그 값이 작다면 사용하지 않은 메모리 공간은 인코딩에서 생략하기
때문입니다.
{{% /alert %}}

이해를 돕기 위해 맵의 키는 모두 문자열을 사용했지만, 어떤 데이터 타입이든 키로
사용할 수 있습니다. 예컨대, `"time"`, `"data"` 문자열 키를 정수 `0`, `1` 로
대체해 페이로드 사이즈를 줄일 수 있습니다.

예제에서 사용된 `xxx_indefinite()` 는 데이터 길이를 지정하지 않습니다. 그렇기
때문에 `cbor_encode_break()` 로 그 끝을 명시해 줘야 합니다. 예제에서는 가장
바깥쪽의 map 길이가 2 이기 때문에 `cbor_encode_map_indefinite()` 대신
`cbor_encode_map(&writer, 2)`을 사용해 `cbor_encode_break()`를 제거할 수
있습니다.

### 디코딩하기
위에서 인코딩한 페이로드를 수신해 아래와 같이 디코딩합니다:

```c
union cbor_value {
	int8_t i8;
	int16_t i16;
	int32_t i32;
	int64_t i64;
	float f32;
	double f64;
	uint8_t *bin;
	char *arr;
	uint8_t arr_copy[MTU];
} val;

cbor_reader_t reader;
cbor_item_t items[MAX_ITEMS];
size_t n;

cbor_reader_init(&reader, items, sizeof(items) / sizeof(*items));
cbor_parse(&reader, received, received_bytes, &n);

for (size_t i = 0; i < n; i++) {
	cbor_item_t *item = items + i;
	cbor_decode(&reader, item, &val, sizeof(val));
}
```

1. 최대 아이템 개수와 함께 reader 를 초기화합니다
	- `cbor_read_init()`
2. 수신한 페이로드를 파싱합니다
	- `cbor_parse()`
3. 파싱한 모든 아이템을 순회하며 디코딩합니다
	- `cbor_decode()`

여러 다른 종류의 데이터 타입을 구분하지 않고 데이터를 다루기 위해 union 을
사용했습니다.

위 라이브러리를 사용한 보다 많은 예제는
[여기](https://github.com/libmcu/cbor/tree/main/examples) 그리고
[여기](https://github.com/libmcu/cbor/tree/main/tests/src)에서 찾을 수
있습니다.

## 나가기
성능이나 메모리 사용량을 고려하면 CBOR 를 사용하지 않을 이유가 없습니다. JSON
과 상호변환도 쉽고요. 하지만 가독성은 다른 무엇보다 주요한 판단 기준이 될 수
있고, JSON 의 저변을 생각하면 일부러 비용을 들여가며 CBOR 로 전환할 필요는 없을
것 같습니다.

마이크로컨트롤러처럼 바이트 단위로 메모리를 관리해야 하는 시스템이거나 네트워크
대역을 아껴야 하는 상황이라면 CBOR 사용을 고려해볼 수 있겠습니다.

## 참고자료
- [RFC8949](https://datatracker.ietf.org/doc/html/rfc8949)
- [cbor.io](https://cbor.io/)
- [CBOR C implementation](https://github.com/libmcu/cbor)
