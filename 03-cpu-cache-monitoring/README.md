# Day 03. CPU Architecture, Cache Memory & System Monitoring

## 1. 학습 목표
- Linux CPU 아키텍처와 논리 CPU 구성 확인
- sysfs와 lscpu를 이용한 CPU 캐시 계층 조사
- 메모리 페이지 크기 확인
- uptime, top, free를 이용한 시스템 모니터링
- CPU 부하 생성 및 프로세스 종료

## 2. 실습 환경

| 항목 | 확인 내용 |
|---|---|
| Host | MacBook Air M2 |
| Virtualization | UTM / QEMU |
| Guest OS | Ubuntu Server ARM64 VM |
| Architecture | aarch64 |
| Logical CPUs | 2 |
| Memory | 약 1.6 GiB 인식 |
| Swap | 약 1.8 GiB |

## 3. CPU 아키텍처

명령어: `lscpu`

- Architecture: aarch64
- CPU op-mode: 64-bit
- CPU(s): 2
- Thread(s) per core: 1
- Core(s) per socket: 2
- Socket(s): 1
- NUMA node(s): 1

VM의 Linux가 AArch64 환경과 논리 CPU 2개를 인식했다. 가상 머신에 노출된 정보이므로 물리 호스트 CPU의 전체 구성을 증명하지는 않는다.

## 4. CPU Cache

조회 경로: `/sys/devices/system/cpu/cpu0/cache/`

| Index | Level | Type | Shared CPUs | Shared CPU Map |
|---|---:|---|---|---|
| index0 | 1 | Data | 0-1 | 3 |
| index1 | 1 | Instruction | 0-1 | 3 |
| index2 | 2 | Unified | 0-1 | 3 |

Shared CPU Map `3`은 2진수 `0011`이며 CPU 0과 CPU 1에 해당하는 비트가 설정된 값이다.

`lscpu -C`에서도 L1d, L1i, L2 계층을 확인했다. 캐시 용량, 연관도, 세트 수, 캐시 라인 관련 세부 값은 출력되지 않았다. sysfs에서도 관련 속성 파일을 확인할 수 없었다.

이는 현재 VM에서 세부 정보가 노출되지 않았다는 의미다. 물리 하드웨어에 해당 속성이 없다는 뜻은 아니다. 이번 출력에서는 L3 항목도 확인되지 않았다.

## 5. Memory Page Size

명령어: `getconf PAGE_SIZE`

결과: `4096 bytes` = `4 KiB`

메모리 페이지 크기와 CPU 캐시 라인 크기는 서로 다른 개념이다. 이번 실습에서는 캐시 라인 크기를 확인하지 못했다.

## 6. System Monitoring

### uptime

관찰 결과:
`load average: 0.08, 0.10, 0.04`

최근 1분, 5분, 15분의 평균 시스템 부하를 나타낸다. Load Average는 단순 CPU 사용률과 동일하지 않으며 실행 대기 및 일부 uninterruptible sleep 상태의 작업도 반영한다.

### top - 부하 생성 전

- CPU Idle: 95.7%
- Memory Total: 1626.8 MiB
- Memory Available: 1316.6 MiB
- Swap Total: 1823.0 MiB
- Swap Used: 0 MiB

### free -h

```text
               total        used        free      shared  buff/cache   available
Mem:           1.6Gi       310Mi       1.0Gi       5.2Mi       355Mi       1.3Gi
Swap:          1.8Gi          0B       1.8Gi
