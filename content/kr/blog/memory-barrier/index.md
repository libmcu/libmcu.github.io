---
date: 2021-10-19
title: 메모리 배리어
description: ARM 기반 CPU memory/synchronization 배리어에 대해 살펴봅니다
author: 권경환 ([@onkwon](https://github.com/onkwon))
tags: ["ARM", "memory barrier"]
---

우선 compiler reordering 과 processor reordering 을 구분해야 할 것 같은데요. 이
글에서는 processor reodering 만 다룹니다. 코드가 실제 CPU 코어에서 동작하기
이전에 컴파일러에 의해 코드가 재배치되는 것을 막는 *컴파일러 메모리 배리어*에
대해서는 따로 언급하지 않습니다.

이 글에서 reodering 은 한글로 재배치라 표현하고 있습니다.

기본적인 메모리 타입과 속성을 알아보고, ARM 아키텍처에서 제공하는 배리어
명령어에 대해 살펴봅니다.

## 배경
CPU 는 프로그램의 인과관계가 깨지지 않는한 MMIO를 포함한 메모리 연산을 자신에게
효율적인 순서로 재배치할 수 있습니다.

멀티 코어에서는 메인 메모리와 별도로 코어별 각각의 레지스터 셋과 캐시를
갖습니다. 여기에서 메모리 가시성visibility 이라고 하는 여러 코어간의 메모리
동기화 문제가 발생합니다. 한 코어에서 발생한 메모리 변경사항이 다른 코어의
시점에 반영되지 않을 수 있기 때문입니다.

이를 해결하기 위해 CPU 아키텍처는 메모리 모델에 따라 타입을 구분하고, 정해진
메모리 ordering 규칙을 따르고 있습니다.

Weak memory model 은 load/store 어떤 조합에서도 명령어가 reordering 될 수 있는
시스템을 말하고, strong memory model 은 제한적인 상황에서만 reordering 이 발생할
수 있는 시스템을 말합니다[^1].

[^1]: ARM 의 경우
  [load-store](https://en.wikipedia.org/wiki/Load%E2%80%93store_architecture)
  아키텍처, 인텔의 경우
  [register-memory](https://en.wikipedia.org/wiki/Register%E2%80%93memory_architecture)
  아키텍처로 각각 weak, strong 시스템입니다.

## 메모리 타입
### Normal Memory
* 일반 코드 및 데이터용 메모리
* 프로세서가 memory reordering 이나 merging 같은 메모리 최적화를 수행할 수 있음

### Device Memory
* load 와 store 가 반드시 코드 순서대로 발생
* 다음과 같은 속성을 지님
    * Gathering
        * 여러개의 접근이 하나의 transaction으로 합쳐질 수 있음
    * Reordering
    * Early write acknowledgement
* 위 속성의 앞글자만 따서 다음과 같은 조합이 만들어질 수 있음
    * !G!R!E = Strongly ordered memory type
    * !G!RE = Device memory type
    * !GRE
    * GRE
 
## 메모리 속성
### Shareable
* 여러 버스 마스터가 있는 시스템에서 마스터끼리 메모리 동기화
    * 가령, 프로세서와 DMA는 하나의 버스를 공유하는 각각의 마스터
* 여러 버스 마스터가 non-shareable 메모리에 접근할 수 있다면, 소프트웨어에서
  데이터 coherency 를 반드시 보장해야 함
* strongly-ordered memory 는 항상 shareable 임

### Cacheable
* Non-cacheable
* Write-Through Cacheable
* Write-Back Cacheable
* Transient
	* 충분한 시간적 지역성temporal locality을 갖지 못하는 경우
	* 다시 사용될 일 없는 데이터에 캐시를 할당하므로써 다른 캐시 엔트리를
	  밀어내지 않도록
	* 캐시의 LRU 위치에 집어넣어 다른 캐시 엔트리에 비해 금방 evict 될 수
	  있도록 할 수 있는 추가적인 속성?

### Shareability Domain
* 시스템을 어떻게 설계하냐에 따라 다르겠지만, 보통 동일한 운영체제를 사용하는
  프로세서들이나 하이퍼바이저는 Inner Shareable shareability domain 에 속함
  - 예컨대, 시스템 내의 두 프로세서는 inner로 묶일 수도 있고 outer로 묶일 수도
    있음
  - 또다른 예로 big.LITTLE 시스템은 두개의 클러스터를 가지는 데, 클러스터 간은
    outer, 클러스터 내는 inner로 묶임
* Non-cacheable 일 경우, 모든 observer(master)에게 coherent 해야 하므로 outer
  shareable 임
* synchronization barriers
  - ISB는 core 별로 실행되어야 하는 명령이므로 제외
  - 외부 버스에 전달되어야 하는 경우(예컨대 DMA): sy 또는 osh
  - inner의 경우: ish.
 
## 메모리 배리어 명령어
* memory barrier 와 synchronization barrier 두가지로 분류
  - DMB는 cpu memory barrier
  - ISB, DSB는 synchronization barrier
* 편의상 4단계 파이프라인을 가정: fetch, decode, execution, write back
* 언급된 아래 배리어외에도 csdb, pssbb, ssbb 가 있음

### ISB
* ISB 명령을 execute 하게 되면 파이프라인을 flush 하고(ISB 이후에 fetch 및
  decode 된 명령들을 취소하고) 새로 fetch 함. 즉, ISB 이전 명령의 모든 연산
  결과가 ISB 이후 명령들에 적용됨
* Control 레지스터 변경이나 인터럽트 활성/비활성 등의 명령 뒤에 사용됨. 해당
  연산의 결과는 다음 명령에 바로 visible 해야 하기 때문

### DMB
* 캐시나 메모리 특성에 따라 의존성이 없는 경우 효율을 위해 메모리 연산을
  프로세서가 reordering 하는 경우가 생기는 데, 이를 방지하기 위해 사용
* 가령 memory mapped I/O 레지스터의 결과가 다음 메모리 접근에 영향을 줄 경우,
  프로세서는 이를 모르고 reordering 할 수 있음. 이때 두 명령 사이에 DMB 명령을
  넣으면 reordering 막아줌

### DSB
* DMB 는 reordering 만을 방지하는 반면, DSB 는 앞선 메모리 연산이 완료될 때 까지
  다음 명령을 실행하지 않는다
* ISB는 flush 하는 반면 DSB는 stall 시킴
* 이를테면, 캐시 invalidate 한 경우 캐시 상태가 시스템 전반에 전달되어야 하기
  때문에 DSB 명령이 사용됨

## 참고자료
* Armv8-M Architecture Reference Manual
