# Day 08. Windows System & Security Monitoring

## 1. Overview

Windows 11 ARM64 가상 머신에서 시스템 리소스, 프로세스, 서비스, 작업 스케줄러, 이벤트 로그 및 엔드포인트 보안 상태를 점검했습니다.

### Environment

| Item | Value |
|---|---|
| Host | MacBook Air M2 |
| Guest OS | Windows 11 Pro |
| Version | 10.0.26300 |
| Build | 26300 |
| Architecture | ARM64 |
| vCPU | 2 |
| Allocated RAM | 4GB |
| Access | Windows PowerShell |

## 2. Objectives

- Windows 시스템 리소스 모니터링
- 프로세스와 서비스 구조 이해
- Task Scheduler 작업 상태 분석
- Windows Event Log 조회 및 해석
- Firewall 및 Microsoft Defender 상태 확인
- 시스템 오류와 보안 이벤트 구분

## 3. System Resource Monitoring

### Task Manager

실제 관찰 결과:

| Resource | Value |
|---|---:|
| CPU Usage | 32% |
| CPU Speed | 3.07GHz |
| Memory | 3.2 / 4.0GB |
| Memory Usage | 80% |
| Disk | 28% |
| Network Receive | 18.3Mbps |
| Processes | 162 |
| Threads | 1,912 |
| Handles | 77,818 |
| Sockets | 1 |
| Virtual Processors | 2 |

메모리 사용률이 80%로 확인됐습니다. 4GB 메모리가 할당된 Windows VM에서 백그라운드 프로세스와 시스템 서비스가 사용하는 리소스를 확인했습니다.

### OS Information

```powershell
Get-CimInstance Win32_OperatingSystem |
Select-Object Caption, Version, BuildNumber, OSArchitecture, LastBootUpTime
```

Actual output:

- Windows 11 Pro
- Version: 10.0.26300
- BuildNumber: 26300
- OSArchitecture: ARM64
- LastBootUpTime: 2026-11-24 21:00:16

## 4. Process Monitoring

```powershell
Get-Process |
Sort-Object CPU -Descending |
Select-Object -First 10 ProcessName, Id, CPU,
@{Name='MemoryMB';Expression={[math]::Round($_.WorkingSet64 / 1MB, 1)}}
```

Actual observation:

| Process | PID | CPU Time | Memory MB |
|---|---:|---:|---:|
| System | 4 | 23.671875 | 0.3 |
| svchost | 3688 | 20.15625 | 333 |
| dwm | 356 | 13.328125 | 80 |
| Taskmgr | 9944 | 11.40625 | 147.8 |
| explorer | 5640 | 11 | 205.2 |
| MicrosoftEdgeUpdate | 9464 | 9.46875 | 46.6 |
| svchost | 5036 | 8.09375 | 76.5 |
| SearchHost | 6320 | 7.71875 | 165.1 |
| svchost | 2600 | 7.625 | 41.1 |
| StartMenuExperienceHost | 6340 | 5.640625 | 156.2 |

### Key Concepts

- `System`: Windows system and kernel-related process
- `svchost.exe`: Windows service hosting process
- `dwm.exe`: Desktop Window Manager
- `explorer.exe`: Windows shell
- `SearchHost.exe`: Windows search interface

**Security Interpretation**

CPU 컬럼은 순간 CPU 사용률이 아니라 프로세스의 누적 CPU 사용 시간(초)입니다.

프로세스 이름만으로 악성 여부를 판단하지 않고 실행 경로, PID, 연결된 서비스, 실행 계정 등을 함께 확인해야 합니다.

## 5. Windows Service Monitoring

```powershell
Get-Service |
Group-Object Status |
Select-Object Name, Count
```

Actual output:

| Status | Count |
|---|---:|
| Running | 103 |
| Stopped | 166 |
| Total | 269 |

실행 중인 서비스 일부:

- AppIDSvc
- Appinfo
- AppXSvc
- AudioEndpointBuilder
- Audiosrv
- BalloonService
- BFE
- BrokerInfrastructure
- camsvc
- CryptSvc

조회 명령:

```powershell
Get-Service |
Where-Object {$_.Status -eq 'Running'} |
Select-Object -First 15 Status, Name, DisplayName
```

초기 명령 입력에서 파이프라인이 잘못 연결되는 문제가 있었습니다. 명령을 분리한 후 정상 조회했습니다.

**Security Interpretation**

Stopped 상태가 반드시 비정상을 의미하지 않습니다. 서비스의 시작 유형, 의존성 및 실제 역할을 함께 확인해야 합니다.

## 6. Task Scheduler Monitoring

```powershell
Get-ScheduledTask |
Group-Object State |
Select-Object Name, Count
```

Actual output:

| State | Count |
|---|---:|
| Running | 6 |
| Ready | 209 |
| Disabled | 35 |
| Total | 250 |

**Security Interpretation**

예약 작업은 정상적인 시스템 유지보수뿐 아니라 공격자의 Persistence 확보에도 악용될 수 있습니다.

확인 대상:
- 실행 파일 경로
- 실행 계정
- 실행 트리거
- 마지막 실행 결과

## 7. Windows Event Log Analysis

### System Log

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='System'
    Level=1,2,3
} -MaxEvents 10 |
Select-Object TimeCreated, Id, LevelDisplayName, ProviderName
```

실제 발견 이벤트:

| Provider | Event ID | Level |
|---|---:|---|
| TPM | 15 | Error |
| Microsoft-Windows-DistributedCOM | 10016 | Warning |

### TPM Event ID 15

상세 메시지:

TPM 장치 드라이버가 TPM 하드웨어에서 복구할 수 없는 오류를 발견했으며, 이로 인해 TPM 서비스(예: 데이터 암호화)를 사용할 수 없다는 내용이 확인됐습니다.

동일한 오류가 반복 발생했습니다.

가상 머신의 TPM 구성이나 드라이버 문제가 원인 후보일 수 있지만 실제 원인은 아직 확정하지 않았습니다.

### DistributedCOM Event ID 10016

확인된 CLSID:

- Windows.SecurityCenter.SecurityAppBroker
- Windows.SecurityCenter.WscBrokerManager
- Windows.SecurityCenter.WscDataProtection

SYSTEM 계정에 대한 COM 서버의 로컬 시작 권한 관련 경고입니다.

단독으로 보안 침해라고 판단하지 않습니다.

### Security Event ID 4672

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4672
} -MaxEvents 3 |
Format-List TimeCreated, Id, Message
```

Actual observation:

| Field | Value |
|---|---|
| Event ID | 4672 |
| Account | SYSTEM |
| SID | S-1-5-18 |
| Domain | NT AUTHORITY |
| Logon ID | 0x3E7 |

확인된 특수 권한:

- SeAssignPrimaryTokenPrivilege
- SeTcbPrivilege
- SeSecurityPrivilege
- SeTakeOwnershipPrivilege
- SeLoadDriverPrivilege
- SeBackupPrivilege
- SeRestorePrivilege
- SeDebugPrivilege
- SeAuditPrivilege
- SeSystemEnvironmentPrivilege
- SeImpersonatePrivilege
- SeDelegateSessionUserImpersonatePrivilege

### Important Security Event IDs

| Event ID | Meaning |
|---|---|
| 4624 | Successful Logon |
| 4625 | Failed Logon |
| 4672 | Special Privileges Assigned to New Logon |

이번 실습에서는 SYSTEM 계정에 대한 4672 이벤트를 확인했습니다. 정상적인 시스템 권한 할당일 수 있으므로 다른 로그인 정보 및 연관 이벤트와 함께 분석해야 합니다.

## 8. Windows Firewall

```powershell
Get-NetFirewallProfile |
Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

Actual output:

| Profile | Enabled | Inbound | Outbound |
|---|---|---|---|
| Domain | True | NotConfigured | NotConfigured |
| Private | True | NotConfigured | NotConfigured |
| Public | True | NotConfigured | NotConfigured |

세 프로필 모두 방화벽이 활성화되어 있었습니다.

NotConfigured는 방화벽 비활성화를 의미하지 않습니다. 개별 규칙과 그룹 정책을 포함한 유효 정책을 추가로 확인해야 합니다.

## 9. Microsoft Defender

```powershell
Get-MpComputerStatus |
Select-Object AMServiceEnabled, AntivirusEnabled, RealTimeProtectionEnabled, AntivirusSignatureLastUpdated
```

Actual output:

| Property | Value |
|---|---|
| AMServiceEnabled | True |
| AntivirusEnabled | True |
| RealTimeProtectionEnabled | True |
| SignatureLastUpdated | 2026-11-24 13:00:09 |

Defender 서비스와 백신, 실시간 보호가 활성화된 것을 확인했습니다.

## 10. Troubleshooting

서비스 상태를 조회하는 과정에서 명령이 잘못 연결됐습니다.

수정 후 정상적으로 결과를 확인했습니다.

```powershell
Get-Service |
Group-Object Status |
Select-Object Name, Count
```

실습 중 명령어의 파이프라인 연결 관계를 확인하는 것이 중요하다는 점을 배웠습니다.

## 11. Key Takeaways

1. Windows 프로세스 및 서비스 구조를 이해했습니다.
2. 작업 스케줄러의 상태별 작업 수를 확인했습니다.
3. TPM 15 및 DCOM 10016 오류·경고를 실제로 발견했습니다.
4. Security Event ID 4672에서 SYSTEM 계정과 특수 권한을 확인했습니다.
5. 방화벽과 Defender의 기본 상태를 점검했습니다.
6. 이벤트 발생과 실제 침해 여부를 구분하는 분석 습관을 익혔습니다.

## 12. CX Operations → TechOps → SecOps

CX Operations에서 SLA와 반복 문의를 관찰하고 문제 원인을 추적했던 것처럼, TechOps와 SecOps에서도 시스템 상태와 이벤트 패턴을 관찰해야 합니다.

| CX Operations | TechOps / SecOps |
|---|---|
| VOC 수집 | 시스템 로그 수집 |
| 반복 문의 분석 | 반복 이벤트 분석 |
| SLA 모니터링 | 시스템 상태 모니터링 |
| 이슈 원인 추적 | 보안 이벤트 조사 |
| SOP 작성 | Incident 대응 절차 문서화 |

## 13. Next Lesson

**Day 09 — Windows Network Monitoring & Connection Analysis**

- Network Adapter
- IP Configuration
- TCP Connection
- Local Port / Remote IP
- PID & Process Mapping
- DNS Resolution
- Routing Table

**Security TechOps Academy — Day 08 Complete.**
