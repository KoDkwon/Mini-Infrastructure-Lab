# Mini Infrastructure Lab

> IDC 환경에서 실제 운영 인프라를 2대의 물리 서버로 재현한 홈랩 프로젝트.
> 네트워크 설계, 방화벽 구성, HA, 모니터링, CI/CD까지 전 과정을 직접 구축.

---

## Overview

실제 운영 환경과 동일한 구조를 소규모로 재현하여 인프라 설계 및 운영 능력을 직접 검증하는 것을 목표로 합니다.

- **기간**: 2026.03 ~ 진행 중
- **목적**: 실무 수준의 인프라 설계/구축/운영 경험 확보
- **핵심 주제**: 가상화, 네트워크 분리(VLAN), 방화벽 정책, HA, 모니터링, CI/CD

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

---

## Stack

| 분류 | 기술 |
|------|------|
| 하이퍼바이저 | XCP-ng 8.x |
| 하이퍼바이저 관리 | Xen Orchestra (XOA) |
| 방화벽/라우터 | OPNsense (Suricata IDS 내장) |
| 네트워크 장비 | Cisco Catalyst C2960 |
| DNS/DHCP | Unbound + ISC DHCP |
| Reverse Proxy | HAProxy |
| Web Server | Nginx |
| Database | PostgreSQL (Primary + Replica) |
| 모니터링 | Prometheus + Grafana |
| SIEM | Wazuh |
| CI/CD | GitLab CE + Jenkins |
| Guest OS | Ubuntu 22.04 LTS |

---

## Hardware

| 항목 | 스펙 |
|------|------|
| 서버 | 2대 (동일 스펙) |
| CPU | Intel Xeon E5-2630v4 × 2소켓 (20코어/40스레드 per 서버) |
| RAM | 64GB DDR4 ECC per 서버 |
| 네트워크 장비 | Cisco Catalyst C2960 × 1대 |
| NIC | 서버당 4포트 (eth0~eth3) |

---

## Network Design

### VLAN 구성

| VLAN ID | 이름 | 서브넷 | 용도 |
|---------|------|--------|------|
| VLAN10 | Management | 192.168.10.0/24 | XCP-ng 호스트 관리 |
| VLAN20 | External/WAN | 192.168.20.0/24 | OPNsense WAN |
| VLAN30 | Internal | 10.10.30.0/24 | 내부 서비스 통신 |
| VLAN40 | DMZ | 172.16.40.0/24 | 외부 노출 서비스 |
| VLAN50 | Storage | 192.168.50.0/24 | NFS/iSCSI, VM 마이그레이션 |
| VLAN60 | Monitoring | 10.10.60.0/24 | 모니터링 전용망 |
| VLAN101 | Uplink | ISP 할당 | 외부 인터넷 통신 |

### NIC 결선

[NODE-01]                         [NODE-02]
eth0 → VLAN10  (Management)       eth0 → VLAN10  (Management)
eth1 → Trunk   (VLAN20,30,40,101) eth1 → Trunk   (VLAN20,30,40,101)
eth2 → VLAN50  (Storage)          eth2 → VLAN50  (Storage)
eth3 → VLAN60  (Monitoring)       eth3 → VLAN60  (Monitoring)

---

## VM 구성

### NODE-01

| VM | 역할 | vCPU | RAM |
|----|------|------|-----|
| VM-01 | OPNsense (방화벽/라우터/VPN/IDS) | 2 | 4GB |
| VM-02 | Xen Orchestra | 2 | 4GB |
| VM-03 | NFS/iSCSI Storage | 2 | 4GB |
| VM-04 | DNS/DHCP | 1 | 2GB |
| VM-05 | HAProxy (Reverse Proxy) | 2 | 2GB |
| VM-06 | Nginx (Web Server) | 4 | 8GB |
| VM-07 | PostgreSQL Primary | 4 | 16GB |

### NODE-02

| VM | 역할 | vCPU | RAM |
|----|------|------|-----|
| VM-08 | Prometheus + Grafana | 2 | 4GB |
| VM-09 | Wazuh (SIEM) | 4 | 8GB |
| VM-10 | GitLab CE | 4 | 8GB |
| VM-11 | Jenkins | 2 | 4GB |
| VM-12 | PostgreSQL Replica | 4 | 8GB |
| VM-13 | Backup Server | 2 | 4GB |

---

## 진행 현황

### Phase 1 - 기반 구축
- [ ] XCP-ng 설치 (NODE-01, NODE-02)
- [ ] XCP-ng Pool 구성 (HA 설정)
- [ ] C2960 VLAN 설정
- [ ] Xen Orchestra 구성
- [ ] NFS 공유 스토리지 구성

### Phase 2 - 네트워크/보안
- [ ] OPNsense 설치 및 VLAN 인터페이스 구성
- [ ] VLAN101 업링크 연결
- [ ] 방화벽 정책 구성
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

#### YYYY-MM-DD | XCP-ng 설치
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

## 트러블슈팅 모음

> Phase 진행 중 발생한 주요 문제와 해결 방법을 별도로 정리합니다.

| 날짜 | 문제 | 원인 | 해결 |
|------|------|------|------|
| - | - | - | - |

---

## 참고 자료

- [XCP-ng 공식 문서](https://xcp-ng.org/docs/)
- [Xen Orchestra 공식 문서](https://xen-orchestra.com/docs/)
- [OPNsense 공식 문서](https://docs.opnsense.org/)
- [Cisco C2960 Configuration Guide](https://www.cisco.com/c/en/us/support/switches/catalyst-2960-series-switches/products-installation-and-configuration-guides-list.html)
- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [Wazuh 공식 문서](https://documentation.wazuh.com/)
