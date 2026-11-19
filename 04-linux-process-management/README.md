

# Day 04. Linux Process Management & Service Monitoring

## 1. 학습 목표

- Linux 프로세스의 PID와 PPID 관계 이해
- `ps`, `pgrep`을 이용한 프로세스 조회
- `ps --forest`를 이용한 프로세스 트리 분석
- R, S, Z 프로세스 상태 관찰
- `kill`을 이용한 프로세스 종료
- `nice`, `renice`를 이용한 CPU 스케줄링 우선순위 조정
- `systemctl`을 이용한 서비스 상태 확인
- `journalctl`을 이용한 시스템 및 보안 로그 분석

## 2. 실습 환경

| 항목 | 확인 내용 |
|---|---|
| Host | MacBook Air M2 |
| Virtualization | UTM / QEMU |
| Guest OS | Ubuntu Server ARM64 VM |
| Architecture | aarch64 |
| Logical CPUs | 2 |
| Memory | 약 1.6 GiB 인식 |
| SSH 접속 | `changseop@192.168.64.3` |

## 3. 프로세스 목록 및 부모·자식 관계

### 프로세스 조회

명령어:

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,comm --sort=-%cpu | head -15
```

PID, PPID, 사용자, 프로세스 상태, CPU 및 메모리 사용량을 기준으로 프로세스 목록을 확인했다.

### SSH 프로세스 계층 분석

명령어:

```bash
ps -o pid,ppid,user,stat,comm -p 1,1460,2833,2892,2893
```

확인 결과:

| PID | PPID | 사용자 | 프로세스 |
|---:|---:|---|---|
| 1 | 0 | root | systemd |
| 1460 | 1 | root | sshd |
| 2833 | 1460 | root | sshd-session |
| 2892 | 2833 | changseop | sshd-session |
| 2893 | 2892 | changseop | bash |

`systemd`에서 시작된 SSH 서버와 SSH 세션, 사용자 셸까지 부모·자식 프로세스 관계를 확인했다.

### 프로세스 트리

명령어:

```bash
ps -eo pid,ppid,stat,comm --forest | head -30
```

`kthreadd`를 부모로 하는 커널 스레드와 `kworker`, `ksoftirqd`, `rcu_preempt`, `kauditd`, `oom_reaper` 등의 커널 작업을 확인했다.

## 4. Linux 프로세스 상태 관찰

### S — Interruptible Sleep

명령어:

```bash
sleep 300 &
ps -o pid,ppid,stat,comm -C sleep
```

확인 결과:

| PID | PPID | STAT | COMMAND |
|---:|---:|---|---|
| 3320 | 2893 | S | sleep |

`S`는 인터럽트 가능한 대기 상태다. `sleep` 프로세스가 지정된 시간 동안 대기하는 것을 확인했다.

### R — Running 또는 Runnable

명령어:

```bash
yes > /dev/null &
ps -eo pid,ppid,stat,comm | grep '[y]es'
```

확인 결과:

```text
3357 2893 R yes
```

`yes` 프로세스가 R 상태로 관찰됐다. R은 실제 CPU에서 실행 중인 상태뿐 아니라 CPU 실행을 기다리는 Runnable 상태도 포함한다.

### Z — Zombie

부모 프로세스가 자식 프로세스의 종료 상태를 회수하지 않는 상황을 재현했다.

첫 번째 터미널에서 부모 프로세스를 실행하고, 두 번째 SSH 터미널에서 자식 프로세스 상태를 확인했다.

```bash
python3 -c 'import os,time; pid=os.fork(); (os._exit(0) if pid == 0 else (print("Child PID:",pid,flush=True),time.sleep(300)))'
```

두 번째 터미널에서 확인한 결과:

```text
PID   PPID  STAT  COMMAND
3450  3449  Z+    python3
```

- PID 3449: 부모 Python 프로세스
- PID 3450: 자식 Python 프로세스
- `Z`: 종료됐지만 부모가 종료 상태를 회수하지 않은 Zombie 상태
- `+`: Foreground Process Group에 속한 상태

자식 프로세스가 먼저 종료됐지만 부모가 대기 중이어서 좀비 상태가 유지되는 것을 확인했다.

프로세스 상태는 실행 여부만으로 판단할 수 없으며, 부모 프로세스의 자식 종료 상태 회수 여부도 중요하다.

### 프로세스 종료

명령어:

```bash
kill 3357
```

확인 결과 `yes` 프로세스가 종료됐고, `sleep` 프로세스는 별도로 대기 상태를 유지했다.

## 5. CPU 스케줄링 우선순위 — nice / renice

### nice로 프로세스 실행

명령어:

```bash
nice -n 10 yes > /dev/null &
```

프로세스 PID를 조회한 뒤 `ps`로 우선순위를 확인했다.

```bash
ps -o pid,ppid,ni,pri,stat,comm -p 3904
```

### renice로 우선순위 변경

명령어:

```bash
renice -n 15 -p 3904
```

실행 결과:

```text
3904 (process ID) old priority 10, new priority 15
```

변경 후 확인:

```text
PID   PPID  NI  PRI  STAT  COMMAND
3904  3553  15  4    RN    yes
```

- 변경 전 NI: 10
- 변경 후 NI: 15
- PRI: 4
- STAT: RN

Nice 값이 10에서 15로 변경된 것을 확인했다.

일반적으로 nice 값이 높아질수록 CPU 스케줄링 우선순위는 낮아진다. 일반 사용자는 자신의 프로세스에 대해 nice 값을 높이는 방향으로 변경할 수 있으며, 우선순위를 높이기 위한 음수 nice 값 설정에는 추가 권한이 필요할 수 있다.

### 프로세스 종료

```bash
kill 3904
```

실행 결과:

```text
[1]+ Terminated nice -n 10 yes > /dev/null
```

우선순위 변경 실습에 사용한 프로세스가 종료된 것을 확인했다.

## 6. systemctl — 서비스 상태 확인

### SSH Socket

명령어:

```bash
systemctl status ssh.socket --no-pager
```

확인 결과:

- Loaded: enabled
- Active: active (running)
- Listen: `0.0.0.0:22`, `[::]:22`
- Triggers: `ssh.service`

SSH Socket이 22번 포트에서 연결을 수신하고 있음을 확인했다.

### SSH Service

명령어:

```bash
systemctl status ssh.service --no-pager
```

확인 결과:

- Loaded: disabled
- Active: active (running)
- TriggeredBy: `ssh.socket`
- Main PID: 1460

SSH Service는 disabled로 표시됐지만 실제 상태는 active였다. 이는 부팅 시 서비스의 직접 자동 시작 설정과 현재 실행 상태가 서로 다른 개념이기 때문이다.

현재 VM에서는 `ssh.socket`이 활성화되어 연결을 수신하고, `ssh.service`가 Socket Activation 구조로 동작하는 것을 확인했다.

### multipathd

명령어:

```bash
systemctl status multipathd --no-pager
```

확인 결과:

- Loaded: enabled
- Active: active (running)
- Main PID: 536
- Status: up

### 실행 중인 서비스 목록

```bash
systemctl list-units --type=service --state=running --no-pager
```

총 19개의 실행 중인 서비스 유닛을 확인했다.

서비스 상태는 개별 유닛 조회뿐 아니라 전체 실행 서비스 목록을 통해서도 확인할 수 있다.

## 7. journalctl — 시스템 및 보안 로그 분석

### SSH Socket 로그

```bash
journalctl -u ssh.socket -n 20 --no-pager
```

SSH Socket의 시작 및 Listening 이벤트를 확인했다.

### SSH Service 로그

```bash
journalctl -u ssh.service -n 20 --no-pager
```

11월 16일 로그에서 SSH 인증 실패 이후 성공한 기록을 확인했다.

```text
08:01:50 password check failed for user (changseop)
08:01:50 authentication failure
08:01:52 Failed password for changseop
08:02:14 Accepted password for changseop
```

인증 실패와 성공 이벤트가 동일한 SSH 연결에서 순서대로 기록된 것을 확인했다.

### Warning 로그

```bash
journalctl -p warning -n 20 --no-pager
```

주요 관찰 내용:

- SCSI 장치 `sr0`의 Device ID 관련 경고
- Wake-on-LAN 설정 미지원
- `cron.service`의 `EXTRA_OPTS` 환경변수 미설정
- `/etc/hosts`의 `security_lab` hostname 형식 오류
- Chrony 시간 동기화 실패
- 시스템 시각이 약 38,585초 차이 난다는 기록

로그의 경고 메시지는 원인과 영향을 추가로 확인해야 하며, 경고 발생 자체만으로 서비스 장애를 확정할 수는 없다.

### 최근 부팅 이후 로그

```bash
journalctl -b -n 30 --no-pager
```

최근 부팅 이후 systemd 서비스 시작 및 종료, PackageKit 실행, AppArmor 접근 통제 이벤트 등을 확인했다.

특히 `who` 프로세스가 `/etc/nsswitch.conf`와 `/etc/passwd`에 접근하려다 AppArmor에 의해 차단된 기록을 확인했다.

### 인증 실패 이벤트 검색

```bash
journalctl -b --no-pager | grep -Ei 'failed password|authentication failure|invalid user'
```

11월 18일 콘솔 로그인 과정에서 발생한 인증 실패 기록을 확인했다.

```text
14:31:00 authentication failure
14:31:03 FAILED LOGIN ... FOR changseop_yeom
14:34:45 authentication failure
14:34:49 FAILED LOGIN ... FOR changseop
```

인증 실패 이벤트의 발생 시각과 대상 계정을 확인했다.

### Error 로그

```bash
journalctl -p err -n 10 --no-pager
```

확인된 주요 메시지:

```text
KHO: Failed to reserve lowmem scratch buffer
PCI: OF: of_root node is NULL, cannot create PCI host bridge node
```

두 메시지는 부팅 시점의 커널 로그에서 확인됐다. 현재 서비스 장애 여부는 별도 상태와 추가 로그를 통해 판단해야 한다.

## 8. SSH 서비스 최종 상태 확인

```bash
systemctl is-active ssh.socket
systemctl is-enabled ssh.socket
```

실행 결과:

```text
active
enabled
```

SSH Socket이 현재 활성 상태이며 부팅 시 자동 활성화되도록 설정되어 있음을 확인했다.

## 9. 핵심 학습 내용

이번 실습에서는 Linux 시스템의 프로세스와 서비스가 어떤 관계로 동작하는지 확인했다.

1. PID와 PPID를 통해 프로세스의 부모·자식 관계를 분석했다.
2. R, S, Z 상태를 직접 재현하고 각 상태의 의미를 확인했다.
3. `nice`와 `renice`로 프로세스의 CPU 스케줄링 우선순위를 변경했다.
4. `systemctl`로 서비스의 현재 상태와 부팅 시 활성화 설정을 구분했다.
5. SSH Socket Activation 구조를 확인했다.
6. `journalctl`로 인증 실패, AppArmor 차단, 시스템 Warning 및 Error 로그를 분석했다.

## 10. 실습 결과 및 다음 학습 주제

프로세스 조회와 상태 분석, 프로세스 우선순위 변경, 서비스 모니터링, 시스템 로그 분석까지 진행했다.

이번 실습에서 확인한 로그는 단순히 오류 메시지를 수집하는 데 그치지 않고, 발생 시각과 프로세스, 서비스, 접근 통제 정책을 함께 살펴봐야 한다는 점을 보여준다.

다음 학습에서는 Linux 파일 권한과 접근 통제, 시스템 로그 및 보안 이벤트 분석을 더 깊게 다룰 예정이다.

