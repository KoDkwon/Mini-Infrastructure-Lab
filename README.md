# 미니 인프라 실습 환경 구성 프로젝트 v6

## 목표
실제 운영 환경과 동일한 네트워크/방화벽/서버 구성을 2대의 물리 서버로 재현

---

## 하드웨어

### 서버 (2대 동일 스펙)

| 항목 | 스펙 |
|------|------|
| CPU | Intel Xeon E5-2630v4 × 2소켓 |
| 코어/스레드 (per 서버) | 20코어 / 40스레드 |
| 기본 클럭 | 2.2GHz (터보 3.1GHz) |
| 캐시 | 25MB Intel Smart Cache × 2 |
| TDP | 85W × 2 = 170W per 서버 |
| RAM | 64GB DDR4 ECC |
| 소켓 | LGA 2011-v3 |
| NIC | 4포트 이상 필요 (eth0~eth3) |
| 스토리지 | HDD 1TB × 8개 per 서버 (RAID10 × 2어레이, 가용 4TB) |
| 총 가용 자원 | 40코어/80스레드, 128GB RAM (2대 합산) |

### 네트워크 장비

| 장비 | 모델 | 수량 | 역할 |
|------|------|------|------|
| L2 스위치 | Cisco Catalyst 2960 | 1대 | 전체 VLAN 트렁킹, 포트 분리 |

**C2960 주요 특성:**
- IEEE 802.1Q 트렁킹 지원 (VLAN 1~4094)
- L2 전용 스위치 → VLAN 간 라우팅은 OPNsense 담당
- EtherChannel (LACP 802.3ad) 지원
- 스패닝트리 (STP/RSTP) 지원

---

## 스토리지 구성

### 물리 디스크 구성 (서버당 동일)

| 항목 | 내용 |
|------|------|
| 디스크 수량 | 1TB HDD × 8개 |
| RAID 구성 | RAID10 × 2어레이 (어레이당 디스크 4개) |
| 어레이당 가용 용량 | 2TB (4TB의 50%) |
| **서버당 총 가용 용량** | **4TB** (md0 2TB + md1 2TB) |
| 내결함성 | 각 어레이 독립 — 어레이당 동일 미러 쌍 아닌 디스크 1개 장애 허용 |
| 구현 방식 | Linux mdadm (소프트웨어 RAID) |

### RAID10 논리 구성

```
[서버당 물리 디스크 8개]

── ARRAY-1 (/dev/md0, 가용 2TB) ──────────────────
  /dev/sda  (1TB) ─┐
  /dev/sdb  (1TB) ─┤─ RAID10 → /dev/md0
  /dev/sdc  (1TB) ─┤
  /dev/sdd  (1TB) ─┘
    Mirror-1: sda ↔ sdb
    Mirror-2: sdc ↔ sdd
    스트라이프: Mirror-1 + Mirror-2

── ARRAY-2 (/dev/md1, 가용 2TB) ──────────────────
  /dev/sde  (1TB) ─┐
  /dev/sdf  (1TB) ─┤─ RAID10 → /dev/md1
  /dev/sdg  (1TB) ─┤
  /dev/sdh  (1TB) ─┘
    Mirror-1: sde ↔ sdf
    Mirror-2: sdg ↔ sdh
    스트라이프: Mirror-1 + Mirror-2
```

### 어레이 용도 분리

| 어레이 | 마운트 | 용도 | 가용 용량 |
|--------|--------|------|----------|
| /dev/md0 | /var/lib/xcp/sr-main | XCP-ng Primary SR (주요 VM 디스크) | 2TB |
| /dev/md1 | /var/lib/xcp/sr-data | XCP-ng Data SR (DB·대용량 데이터·백업) | 2TB |

### VM별 디스크 할당

#### NODE-01 (md0: Primary SR 2TB / md1: Data SR 2TB)

**md0 — Primary SR (VM OS + 경량 데이터)**

| VM | 역할 | OS 디스크 | 데이터 디스크 | 소계 |
|----|------|----------|------------|------|
| VM-01 | OPNsense | 20GB | 30GB (로그/IDS 룰) | 50GB |
| VM-02 | Xen Orchestra | 20GB | 30GB | 50GB |
| VM-03 | NFS/iSCSI Storage Server | 20GB | 50GB (OS측 메타데이터) | 70GB |
| VM-04 | DNS/DHCP | 20GB | 10GB | 30GB |
| VM-05 | HAProxy | 20GB | 30GB (로그) | 50GB |
| VM-06 | Web Server (Nginx) | 20GB | 100GB (콘텐츠/로그) | 120GB |
| XCP-ng OS / 스냅샷 버퍼 | — | 100GB | 400GB | 500GB |
| **md0 합계** | | | | **≈ 870GB / 2,000GB** |

**md1 — Data SR (대용량 데이터)**

| VM | 역할 | 데이터 디스크 | 소계 |
|----|------|------------|------|
| VM-03 | NFS 공유 스토리지 풀 | 800GB | 800GB |
| VM-07 | DB Primary (PostgreSQL) | 800GB | 800GB |
| 예비/확장 버퍼 | — | 400GB | 400GB |
| **md1 합계** | | | **≈ 2,000GB / 2,000GB** |

#### NODE-02 (md0: Primary SR 2TB / md1: Data SR 2TB)

**md0 — Primary SR (VM OS + 경량 데이터)**

| VM | 역할 | OS 디스크 | 데이터 디스크 | 소계 |
|----|------|----------|------------|------|
| VM-08 | Prometheus + Grafana | 20GB | 200GB (메트릭 90일 보존) | 220GB |
| VM-10 | GitLab CE | 20GB | 300GB (repo/CI artifacts) | 320GB |
| VM-11 | Jenkins | 20GB | 150GB (빌드 아티팩트) | 170GB |
| XCP-ng OS / 스냅샷 버퍼 | — | 100GB | 300GB | 400GB |
| **md0 합계** | | | | **≈ 1,110GB / 2,000GB** |

**md1 — Data SR (대용량 데이터)**

| VM | 역할 | 데이터 디스크 | 소계 |
|----|------|------------|------|
| VM-09 | Wazuh SIEM | 600GB (이벤트/인덱스 180일) | 600GB |
| VM-12 | DB Replica (PostgreSQL) | 800GB (Primary와 동일) | 800GB |
| VM-13 | Backup Server | 500GB (백업 대상) | 500GB |
| 예비 버퍼 | — | 100GB | 100GB |
| **md1 합계** | | | **≈ 2,000GB / 2,000GB** |

### mdadm RAID10 구성 명령어 (참고)

```bash
# ── ARRAY-1 생성 ──────────────────────────────────────
mdadm --create /dev/md0 \
  --level=10 \
  --raid-devices=4 \
  /dev/sda /dev/sdb /dev/sdc /dev/sdd

# ── ARRAY-2 생성 ──────────────────────────────────────
mdadm --create /dev/md1 \
  --level=10 \
  --raid-devices=4 \
  /dev/sde /dev/sdf /dev/sdg /dev/sdh

# 파일시스템 생성
mkfs.xfs /dev/md0
mkfs.xfs /dev/md1

# 마운트 포인트 등록
mkdir -p /var/lib/xcp/sr-main /var/lib/xcp/sr-data
echo '/dev/md0 /var/lib/xcp/sr-main xfs defaults,nofail 0 0' >> /etc/fstab
echo '/dev/md1 /var/lib/xcp/sr-data xfs defaults,nofail 0 0' >> /etc/fstab

# RAID 상태 확인
cat /proc/mdstat
mdadm --detail /dev/md0
mdadm --detail /dev/md1

# mdadm 설정 저장 (재부팅 후 자동 인식)
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

### 스토리지 용량 요약

| 구분 | NODE-01 | NODE-02 |
|------|---------|---------|
| 물리 디스크 | 1TB × 8 = 8TB | 1TB × 8 = 8TB |
| md0 (Primary SR) 가용 | 2TB | 2TB |
| md1 (Data SR) 가용 | 2TB | 2TB |
| **서버당 총 가용** | **4TB** | **4TB** |
| md0 사용 / 여유 | ~870GB / ~1,130GB | ~1,110GB / ~890GB |
| md1 사용 / 여유 | ~1,600GB / ~400GB | ~1,900GB / ~100GB |
| **전체 합산 가용** | | **8TB (2대 합산)** |

> **주의:** NODE-02 md1은 여유가 약 100GB로 빠듯합니다. Wazuh 인덱스 보존 기간(현재 180일) 단축 또는 Backup Server 대상 정책 조정을 권장합니다.

---

## VLAN 구성

| VLAN ID | 이름 | 서브넷 | 용도 |
|---------|------|--------|------|
| VLAN10 | Management | 192.168.10.0/24 | XCP-ng 호스트 관리 |
| VLAN20 | External/WAN | 192.168.20.0/24 | OPNsense WAN 인터페이스 |
| VLAN30 | Internal | 10.10.30.0/24 | 내부 서비스 통신 |
| VLAN40 | DMZ | 172.16.40.0/24 | 외부 노출 서비스 |
| VLAN50 | Storage/Migration | 192.168.50.0/24 | NFS/iSCSI, VM 마이그레이션 |
| VLAN60 | Monitoring | 10.10.60.0/24 | 모니터링 전용망 |
| VLAN101 | Uplink | (ISP 할당 또는 공인 IP 대역) | 인터넷/외부 통신 업링크 |

---

## 네트워크 트래픽 흐름

```
[인터넷/외부망]
       │
       │ (공인 IP or ISP 회선)
       ▼
[외부 라우터/모뎀]
       │
       │ VLAN101 (Uplink)
       ▼
[Cisco C2960 스위치 - 1대]
       │
       ├── VLAN101 → OPNsense WAN (외부 통신 진입점)
       ├── VLAN10  → XCP-ng Management
       ├── VLAN30  → 내부 서비스망
       ├── VLAN40  → DMZ
       ├── VLAN50  → Storage/Migration
       └── VLAN60  → Monitoring
               │
               ▼
        [OPNsense VM]
         WAN / LAN / DMZ
               │
       ┌───────┴────────┐
       ▼                ▼
  [내부 서비스]        [DMZ]
  Web, DB, CI/CD      Nginx, HAProxy
```

---

## 물리 NIC 결선

### 서버당 NIC 4포트 구성

```
[NODE-01]
eth0 (1GbE) ── C2960 Gi0/2 ── VLAN10  Access  (Management)
eth1 (1GbE) ── C2960 Gi0/3 ── Trunk           (VLAN20,30,40,101)
eth2 (1GbE) ── C2960 Gi0/4 ── VLAN50  Access  (Storage/Migration)
eth3 (1GbE) ── C2960 Gi0/5 ── VLAN60  Access  (Monitoring)

[NODE-02]
eth0 (1GbE) ── C2960 Gi0/6 ── VLAN10  Access  (Management)
eth1 (1GbE) ── C2960 Gi0/7 ── Trunk           (VLAN20,30,40,101)
eth2 (1GbE) ── C2960 Gi0/8 ── VLAN50  Access  (Storage/Migration)
eth3 (1GbE) ── C2960 Gi0/9 ── VLAN60  Access  (Monitoring)
```

---

## C2960 포트 배분 (24포트 기준)

```
[업링크]
Gi0/1  - 외부 라우터/모뎀 연결 (VLAN101 Access)

[NODE-01]
Gi0/2  - NODE-01 eth0  (VLAN10  Access - Management)
Gi0/3  - NODE-01 eth1  (Trunk: VLAN20,30,40,101)
Gi0/4  - NODE-01 eth2  (VLAN50  Access - Storage)
Gi0/5  - NODE-01 eth3  (VLAN60  Access - Monitoring)

[NODE-02]
Gi0/6  - NODE-02 eth0  (VLAN10  Access - Management)
Gi0/7  - NODE-02 eth1  (Trunk: VLAN20,30,40,101)
Gi0/8  - NODE-02 eth2  (VLAN50  Access - Storage)
Gi0/9  - NODE-02 eth3  (VLAN60  Access - Monitoring)

[예비/확장]
Gi0/10~24 - 추후 확장용 예비 포트 (15포트)
```

### C2960 설정 명령어 (참고)

```
! VLAN 생성
vlan 10
 name Management
vlan 20
 name External-WAN
vlan 30
 name Internal
vlan 40
 name DMZ
vlan 50
 name Storage-Migration
vlan 60
 name Monitoring
vlan 101
 name Uplink-External

! 업링크 포트 (Gi0/1) - VLAN101 Access
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 101
 description Uplink-to-Router

! NODE-01 Management (Gi0/2)
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 10
 description NODE-01-Management

! NODE-01 서비스망 Trunk (Gi0/3)
interface GigabitEthernet0/3
 switchport mode trunk
 switchport trunk allowed vlan 20,30,40,101
 description NODE-01-Service-Trunk

! NODE-01 Storage (Gi0/4)
interface GigabitEthernet0/4
 switchport mode access
 switchport access vlan 50
 description NODE-01-Storage

! NODE-01 Monitoring (Gi0/5)
interface GigabitEthernet0/5
 switchport mode access
 switchport access vlan 60
 description NODE-01-Monitoring

! NODE-02 Management (Gi0/6)
interface GigabitEthernet0/6
 switchport mode access
 switchport access vlan 10
 description NODE-02-Management

! NODE-02 서비스망 Trunk (Gi0/7)
interface GigabitEthernet0/7
 switchport mode trunk
 switchport trunk allowed vlan 20,30,40,101
 description NODE-02-Service-Trunk

! NODE-02 Storage (Gi0/8)
interface GigabitEthernet0/8
 switchport mode access
 switchport access vlan 50
 description NODE-02-Storage

! NODE-02 Monitoring (Gi0/9)
interface GigabitEthernet0/9
 switchport mode access
 switchport access vlan 60
 description NODE-02-Monitoring

! 스위치 Management SVI
interface Vlan10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
```

---

## IP 할당

### Uplink (VLAN101)
- ISP 또는 외부 라우터에서 할당받은 공인/사설 IP
- OPNsense WAN 인터페이스가 이 대역에서 IP 취득 (DHCP 또는 고정)

### Management (VLAN10)
- 192.168.10.1  - C2960 SVI
- 192.168.10.10 - NODE-01 (XCP-ng Host)
- 192.168.10.11 - NODE-02 (XCP-ng Host)
- 192.168.10.20 - Xen Orchestra

### Internal (VLAN30)
- 10.10.30.1    - OPNsense LAN 인터페이스 (게이트웨이)
- 10.10.30.10   - DNS/DHCP 서버
- 10.10.30.11   - Reverse Proxy (HAProxy)
- 10.10.30.20   - Web Server
- 10.10.30.21   - DB Primary (PostgreSQL)
- 10.10.30.22   - DB Replica (PostgreSQL)

### DMZ (VLAN40)
- 172.16.40.1   - OPNsense DMZ 인터페이스
- 172.16.40.10  - Web Server (공개)
- 172.16.40.11  - Reverse Proxy

### Storage (VLAN50)
- 192.168.50.10 - NODE-01 Storage Interface
- 192.168.50.11 - NODE-02 Storage Interface
- 192.168.50.20 - NFS/iSCSI Storage Server

### Monitoring (VLAN60)
- 10.10.60.1    - 게이트웨이 (OPNsense OPT 인터페이스)
- 10.10.60.10   - Prometheus + Grafana
- 10.10.60.11   - Wazuh SIEM

---

## VM 구성

### NODE-01 (20코어/40스레드, 64GB RAM)

| VM | 역할 | vCPU | RAM | 네트워크 |
|----|------|------|-----|---------|
| VM-01 | OPNsense (방화벽/라우터/VPN/IDS) | 2 | 4GB | VLAN101, 30, 40 |
| VM-02 | Xen Orchestra | 2 | 4GB | VLAN10 |
| VM-03 | NFS/iSCSI Storage Server | 2 | 4GB | VLAN50 |
| VM-04 | DNS/DHCP | 1 | 2GB | VLAN30 |
| VM-05 | Reverse Proxy (HAProxy) | 2 | 2GB | VLAN30, 40 |
| VM-06 | Web Server (Nginx) | 4 | 8GB | VLAN30, 40 |
| VM-07 | DB Primary (PostgreSQL) | 4 | 16GB | VLAN30 |

### NODE-02 (20코어/40스레드, 64GB RAM)

| VM | 역할 | vCPU | RAM | 네트워크 |
|----|------|------|-----|---------|
| VM-08 | 모니터링 (Prometheus + Grafana) | 2 | 4GB | VLAN60 |
| VM-09 | SIEM (Wazuh) | 4 | 8GB | VLAN60 |
| VM-10 | GitLab CE | 4 | 8GB | VLAN30 |
| VM-11 | Jenkins | 2 | 4GB | VLAN30 |
| VM-12 | DB Replica (PostgreSQL) | 4 | 8GB | VLAN30 |
| VM-13 | Backup Server | 2 | 4GB | VLAN50 |

---

## 방화벽 (OPNsense)

### 인터페이스 구성

| OPNsense 인터페이스 | 연결 VLAN | 역할 |
|--------------------|----------|------|
| WAN | VLAN101 | 외부 인터넷 통신 (업링크) |
| LAN | VLAN30 | 내부 서비스망 |
| DMZ | VLAN40 | 외부 노출 서비스 |
| OPT1 | VLAN60 | 모니터링 전용망 |

### 기본 방화벽 정책

```
[WAN(VLAN101) → DMZ]
ALLOW  TCP  ANY → 172.16.40.10  :80,443   (HTTP/HTTPS)
DENY   ANY  ANY → ANY                     (Default Deny)

[DMZ → Internal]
ALLOW  TCP  172.16.40.0/24 → 10.10.30.21  :5432  (Web→DB만 허용)
DENY   ANY  DMZ → Internal

[Internal → WAN]
ALLOW  TCP  10.10.30.0/24 → ANY  :80,443
ALLOW  TCP  10.10.30.0/24 → ANY  :22
DENY   ANY  ANY → ANY

[Monitoring → Internal]
ALLOW  TCP  10.10.60.0/24 → 10.10.30.0/24  :9100  (Node Exporter)
ALLOW  TCP  10.10.60.0/24 → 10.10.30.0/24  :514   (Syslog)
DENY   ANY  Monitoring → ANY               (그 외 차단)

[Management → ANY]
ALLOW  TCP  192.168.10.0/24 → ANY  :22,443
```

---

## 주요 소프트웨어 스택

| 역할 | 소프트웨어 |
|------|-----------|
| 하이퍼바이저 | XCP-ng 8.x |
| 하이퍼바이저 관리 | Xen Orchestra (XOA) |
| 방화벽/라우터/IDS/VPN | OPNsense (Suricata 내장) |
| DNS/DHCP | Unbound + ISC DHCP |
| Reverse Proxy | HAProxy |
| Web Server | Nginx |
| Database | PostgreSQL (Primary + Replica) |
| 모니터링 | Prometheus + Grafana |
| SIEM | Wazuh |
| CI/CD | GitLab CE + Jenkins |
| 백업 | Xen Orchestra 내장 백업 |
| Guest OS | Ubuntu 22.04 LTS |
| RAID 관리 | Linux mdadm (소프트웨어 RAID10) |

---

## 구축 단계

| Phase | 내용 | 상태 |
|-------|------|------|
| Phase 1 | XCP-ng 설치, RAID10 구성, Pool 구성, C2960 VLAN 설정, Xen Orchestra | 미시작 |
| Phase 2 | OPNsense 방화벽 구성 (VLAN101 업링크 포함), DNS/DHCP, Reverse Proxy | 미시작 |
| Phase 3 | 서비스 VM 배포 (Web, DB, HAProxy), HA 적용 | 미시작 |
| Phase 4 | 모니터링, SIEM, CI/CD, 백업 구성 | 미시작 |

---

## 변경 이력

| 버전 | 변경 내용 |
|------|---------|
| v1 | 최초 설계 (E5-2670 × 2, pfSense, Suricata 별도 VM) |
| v2 | CPU → E5-2630v4 × 2소켓 / 방화벽 → OPNsense / Suricata VM 제거 / C2960 추가 |
| v3 | 스위치 1대 확정 / VLAN101 외부 업링크 추가 / C2960 포트 배분 및 설정 명령어 추가 |
| v4 | 서버당 NIC eth3 추가 / VLAN50(Storage), VLAN60(Monitoring) 포트 누락 수정 / C2960 설정 명령어 전체 포트 완성 / Monitoring 방화벽 정책 추가 |
| v5 | 스토리지 구성 추가 / 서버당 1TB HDD × 4 → RAID10 단일 어레이 (가용 2TB) / VM별 디스크 할당 상세화 / mdadm 설정 명령어 추가 |
| v6 | 스토리지 확장 / 서버당 1TB HDD × 8 → RAID10 × 2어레이 (md0 Primary SR 2TB + md1 Data SR 2TB, 총 가용 4TB) / 어레이 용도 분리 / VM 디스크 할당 재배분 / mdadm 이중 어레이 명령어 업데이트 |
