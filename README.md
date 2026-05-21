# Arista Network Auto-Check System

> Kubernetes 위에서 운영되는 Arista 스위치 자동 상태 점검 시스템.  
> Python SSH + Ansible EOS Collection을 활용하여 네트워크 장비의 상태를 자동으로 수집·분석·보고.

---

## 개요

네트워크 엔지니어가 수동으로 반복하던 장비 상태 점검 업무를 자동화한 시스템입니다.  
Kubernetes 위에서 동작하며, ConfigMap을 활용한 **무중단 설정 변경(Live Update)** 을 지원합니다.

## 주요 기능

- **자동 상태 수집**: CPU, 메모리, 온도, 팬, PSU 등 장비 헬스 데이터 자동 수집
- **멀티 장비 동시 점검**: device_list 기반 다수 스위치 병렬 점검
- **자동 리포트 생성**: 수집 결과를 정형화된 보고서로 자동 출력
- **Live Update**: 재배포 없이 ConfigMap 수정만으로 장비 목록·명령어·플레이북 즉시 반영

## 아키텍처

```
Kubernetes Cluster
└── Namespace: ansible
    ├── Deployment (ansible-manage-node)
    │   └── Container: auto-check (Python + Ansible)
    ├── Service (HTTP :8080)
    └── ConfigMaps (Live Update)
        ├── ansible.cfg         ← Ansible 설정
        ├── hosts (inventory)   ← 장비 목록
        ├── auto-check.yaml     ← Ansible Playbook
        ├── device_list.txt     ← SSH 대상 장비
        ├── commands.txt        ← 실행 명령어
        └── auto-check.py       ← Python SSH 스크립트
```

## 수집 항목

| 항목 | 수집 데이터 |
|---|---|
| 리소스 | CPU 사용률 (user/sys), 메모리 사용률 |
| 환경 | 온도 상태, 쿨링 상태, 팬 상태 |
| 전원 | PSU 상태 (Ok / Power Loss / fault) |
| 장비 정보 | EOS 버전, 모델명, 업타임 |
| 라우팅 | `show ip route`, `show ip int b` |
| 로그 | 최근 로그 100줄 |

## 기술 스택

| 항목 | 기술 |
|---|---|
| 인프라 | Kubernetes |
| 자동화 | Ansible (`arista.eos` Collection) |
| 스크립트 | Python 3 + Paramiko (SSH) |
| 컨테이너 | Docker → GHCR (`ghcr.io/zpzg333/auto-check`) |
| 설정 관리 | Kubernetes ConfigMap (Live Update) |

## 사용 기술 포인트

- **Ansible EOS Collection**: `arista.eos.eos_command`로 Arista 전용 명령 실행 및 JSON 파싱
- **Jinja2 정규식 필터**: playbook 내 `regex_search`로 CLI 출력 데이터 추출·가공
- **Kubernetes 운영**: ConfigMap 기반 설정 분리로 컨테이너 재시작 없이 운영 파라미터 변경
- **멀티 레이어 자동화**: Python SSH(paramiko)와 Ansible을 혼용하여 상황에 맞게 활용

## 실행 방법

```bash
# 네임스페이스 생성
kubectl apply -f namespace.yaml

# ConfigMap 배포 (설정 주입)
kubectl apply -f cm-2-ansible-cfg.yaml
kubectl apply -f cm-2-host.yaml
kubectl apply -f cm-2-playbook.yaml
kubectl apply -f cm-autocheckpy.yaml
kubectl apply -f cm-command.yaml
kubectl apply -f cm-devicelist.yaml

# 애플리케이션 배포
kubectl apply -f deploy.yaml
kubectl apply -f svc.yaml
```

## 배경

Arista 스위치 장비 점검 업무를 자동화하여 반복 작업을 줄이고,  
이상 징후를 조기에 탐지할 수 있도록 구성한 실무 프로젝트입니다.
