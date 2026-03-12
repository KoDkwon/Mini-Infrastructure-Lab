# Mini Infrastructure Lab

> IDC 환경에서 실제 운영 인프라를 2대의 물리 서버로 재현한 홈랩 프로젝트.
> 네트워크 설계, 방화벽 구성, HA, 모니터링, CI/CD까지 전 과정을 직접 구축.

---

## Overview

실제 운영 환경과 동일한 구조를 소규모로 재현하여 인프라 설계 및 운영 능력을 직접 검증하는 것을 목표로 합니다.

- **기간**: 2024.03 ~ 진행 중
- **목적**: 실무 수준의 인프라 설계/구축/운영 경험 확보
- **핵심 주제**: 가상화, 네트워크 분리(VLAN), 방화벽 정책, HA, 스토리지, 모니터링, CI/CD

---

## Architecture

```
[인터넷/외부망]
      │
[외부 라우터/모뎀]
      │ VLAN101 (Uplink)
      ▼
[Cisco Catalyst C2960 - L2 Switch]
      │
      ├── VLAN10  (192.168.10.0/24) - Management
      ├── VLAN20  (192.168.20.0/24) - External/WAN
      ├── VLAN30  (10.10.30.0/24)   - Internal
      ├── VLAN40  (172.16.40.0/24)  - DMZ
      ├── VLAN50  (192.168.50.0/24) - Storage/Migration
      ├── VLAN60  (10.10.60.0/24)   - Monitoring
      └── VLAN101 (ISP 할당)        - Uplink
            │
      ┌─────┴──────┐
      ▼            ▼
  [NODE-01]    [NODE-02]
  XCP-ng       XCP-ng
  Pool Master  Pool Slave
      │            │
      └─────HA─────┘
         (VLAN50)
```

---

## Stack

| 분류 | 기술 |
|------|------|
| 하이퍼바이저 | XCP-ng 8.x |
| 하이퍼바이저 관리 | Xen Orchestra (XOA) |
| 방화벽/라우터/IDS/VPN | OPNsense (Suricata 내장) |
| 네트워크 장비 | Cisco Catalyst C2960 |
| DNS/DHCP | Unbound + ISC DHCP |
| Reverse Proxy | HAProxy |
| Web Server | Nginx |
| Database | PostgreSQL (Primary + Replica) |
| 모니터링 | Prometheus + Grafana |
| SIEM | Wazuh |
| CI/CD | GitLab CE + Jenkins |
| RAID 관리 | Linux mdadm (소프트웨어 RAID10) |
| Guest OS | Ubuntu 22.04 LTS |

---

## Hardware

| 항목 | 스펙 |
|------|------|
| 서버 | 2대 (동일 스펙) |
| CPU | Intel Xeon E5-2630v4 × 2소켓 (20코어/40스레드 per 서버) |
| RAM | 64GB DDR4 ECC per 서버 |
| 스토리지 | 1TB HDD × 8개 per 서버 (RAID10 × 2어레이, 가용 4TB) |
| NIC | 서버당 4포트 (eth0~eth3) |
| 네트워크 장비 | Cisco Catalyst C2960 × 1대 |
| 총 가용 자원 | 40코어/80스레드, 128GB RAM, 8TB (2대 합산) |

---

## Storage Design

### RAID 구성 (서버당 동일)

```
[물리 디스크 8개]

── ARRAY-1 (/dev/md0, 가용 2TB) ─────────────
  /dev/sda ~ /dev/sdd → RAID10 → Primary SR
  마운트: /var/lib/xcp/sr-main
  용도: VM OS 디스크, 경량 데이터

── ARRAY-2 (/dev/md1, 가용 2TB) ─────────────
  /dev/sde ~ /dev/sdh → RAID10 → Data SR
  마운트: /var/lib/xcp/sr-data
  용도: DB 데이터, NFS 공유 풀, 백업
```

### 용량 요약

| 구분 | NODE-01 | NODE-02 |
|------|---------|---------|
| md0 (Primary SR) | 2TB / 사용 ~870GB | 2TB / 사용 ~1,110GB |
| md1 (Data SR) | 2TB / 사용 ~1,600GB | 2TB / 사용 ~1,900GB |
| **서버당 총 가용** | **4TB** | **4TB** |
| **전체 합산** | | **8TB** |

> NODE-02 md1 여유 약 100GB로 빠듯. Wazuh 인덱스 보존 기간 단축 또는 백업 정책 조정 필요.

---

## Network Design

### VLAN 구성

| VLAN ID | 이름 | 서브넷 | 용도 |
|---------|------|--------|------|
| VLAN10 | Management | 192.168.10.0/24 | XCP-ng 호스트 관리 |
| VLAN20 | External/WAN | 192.168.20.0/24 | OPNsense WAN |
| VLAN30 | Internal | 10.10.30.0/24 | 내부 서비스 통신 |
| VLAN40 | DMZ | 172.16.40.0/24 | 외부 노출 서비스 |
| VLAN50 | Storage/Migration | 192.168.50.0/24 | NFS/iSCSI, VM 마이그레이션 |
| VLAN60 | Monitoring | 10.10.60.0/24 | 모니터링 전용망 |
| VLAN101 | Uplink | ISP 할당 | 외부 인터넷 통신 |

### NIC 결선

```
[NODE-01]                              [NODE-02]
eth0 → VLAN10  Access (Management)     eth0 → VLAN10  Access (Management)
eth1 → Trunk   (VLAN20,30,40,101)      eth1 → Trunk   (VLAN20,30,40,101)
eth2 → VLAN50  Access (Storage)        eth2 → VLAN50  Access (Storage)
eth3 → VLAN60  Access (Monitoring)     eth3 → VLAN60  Access (Monitoring)
```

### IP 할당

| VLAN | 주요 IP |
|------|---------|
| VLAN10 | .1 C2960 SVI / .10 NODE-01 / .11 NODE-02 / .20 Xen Orchestra |
| VLAN30 | .1 OPNsense LAN / .10 DNS / .11 HAProxy / .20 Web / .21 DB Primary / .22 DB Replica |
| VLAN40 | .1 OPNsense DMZ / .10 Web(공개) / .11 Reverse Proxy |
| VLAN50 | .10 NODE-01 / .11 NODE-02 / .20 NFS Storage |
| VLAN60 | .1 OPNsense OPT1 / .10 Prometheus+Grafana / .11 Wazuh |

---

## VM 구성

### NODE-01 (20코어/40스레드, 64GB RAM)

| VM | 역할 | vCPU | RAM | 네트워크 |
|----|------|------|-----|---------|
| VM-01 | OPNsense (방화벽/라우터/VPN/IDS) | 2 | 4GB | VLAN101, 30, 40 |
| VM-02 | Xen Orchestra | 2 | 4GB | VLAN10 |
| VM-03 | NFS/iSCSI Storage Server | 2 | 4GB | VLAN50 |
| VM-04 | DNS/DHCP | 1 | 2GB | VLAN30 |
| VM-05 | HAProxy (Reverse Proxy) | 2 | 2GB | VLAN30, 40 |
| VM-06 | Nginx (Web Server) | 4 | 8GB | VLAN30, 40 |
| VM-07 | PostgreSQL Primary | 4 | 16GB | VLAN30 |

### NODE-02 (20코어/40스레드, 64GB RAM)

| VM | 역할 | vCPU | RAM | 네트워크 |
|----|------|------|-----|---------|
| VM-08 | Prometheus + Grafana | 2 | 4GB | VLAN60 |
| VM-09 | Wazuh (SIEM) | 4 | 8GB | VLAN60 |
| VM-10 | GitLab CE | 4 | 8GB | VLAN30 |
| VM-11 | Jenkins | 2 | 4GB | VLAN30 |
| VM-12 | PostgreSQL Replica | 4 | 8GB | VLAN30 |
| VM-13 | Backup Server | 2 | 4GB | VLAN50 |

---

## Firewall Policy (OPNsense)

| OPNsense 인터페이스 | 연결 VLAN | 역할 |
|--------------------|----------|------|
| WAN | VLAN101 | 외부 인터넷 통신 |
| LAN | VLAN30 | 내부 서비스망 |
| DMZ | VLAN40 | 외부 노출 서비스 |
| OPT1 | VLAN60 | 모니터링 전용망 |

```
[WAN → DMZ]              ALLOW TCP ANY       → 172.16.40.10  :80,443
[DMZ → Internal]         ALLOW TCP DMZ       → DB Primary    :5432 only
[Internal → WAN]         ALLOW TCP Internal  → ANY           :80,443,22
[Monitoring → Internal]  ALLOW TCP VLAN60    → VLAN30        :9100,514
[Management → ANY]       ALLOW TCP VLAN10    → ANY           :22,443
                         DENY  ALL others
```

---

## 진행 현황

### Phase 1 - 기반 구축
- [ ] XCP-ng 설치 (NODE-01, NODE-02)
- [ ] mdadm RAID10 × 2어레이 구성 (md0, md1)
- [ ] XCP-ng SR 등록 (Primary SR / Data SR)
- [ ] XCP-ng Pool 구성 및 HA 설정
- [ ] C2960 VLAN 설정
- [ ] Xen Orchestra 구성
- [ ] NFS 공유 스토리지 구성

### Phase 2 - 네트워크/보안
- [ ] OPNsense 설치 및 VLAN 인터페이스 구성
- [ ] VLAN101 업링크 연결
- [ ] 방화벽 정책 구성
- [ ] Suricata IDS 활성화
- [ ] DNS/DHCP 구성
- [ ] HAProxy Reverse Proxy 구성

### Phase 3 - 서비스 배포
- [ ] Nginx Web Server 구성
- [ ] PostgreSQL Primary/Replica 구성
- [ ] HA 동작 테스트 (NODE 강제 다운 → VM 자동 복구)
- [ ] Live Migration 테스트

### Phase 4 - 운영 체계
- [ ] Prometheus + Grafana 구성
- [ ] Wazuh SIEM 구성
- [ ] GitLab CE 구성
- [ ] Jenkins CI/CD 파이프라인 구성
- [ ] 백업 정책 구성

---

## 진행 일지

### [Phase 1] 기반 구축

#### YYYY-MM-DD | XCP-ng 설치 및 RAID 구성
**시도한 것**
- 내용 작성

**발생한 문제**
- 내용 작성

**원인**
- 내용 작성

**해결 방법**
```bash
# 관련 명령어 기록
```

**배운 것**
- 내용 작성

---

### [Phase 2] 네트워크/보안

#### YYYY-MM-DD | OPNsense 구성
> 일지 추가 예정

---

### [Phase 3] 서비스 배포

#### YYYY-MM-DD | PostgreSQL HA 구성
> 일지 추가 예정

---

### [Phase 4] 운영 체계

#### YYYY-MM-DD | Grafana 대시보드 구성
> 일지 추가 예정

---

## 🚨 트러블슈팅 모음

| 날짜 | Phase | 문제 | 원인 | 해결 |
|------|-------|------|------|------|
| - | - | - | - | - |

---

## 📚 참고 자료

- [XCP-ng 공식 문서](https://xcp-ng.org/docs/)
- [Xen Orchestra 공식 문서](https://xen-orchestra.com/docs/)
- [OPNsense 공식 문서](https://docs.opnsense.org/)
- [Cisco C2960 Configuration Guide](https://www.cisco.com/c/en/us/support/switches/catalyst-2960-series-switches/products-installation-and-configuration-guides-list.html)
- [mdadm RAID 관리](https://linux.die.net/man/8/mdadm)
- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [Wazuh 공식 문서](https://documentation.wazuh.com/)
