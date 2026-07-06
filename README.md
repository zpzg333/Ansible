# Arista Network Auto-Check System

> Kubernetes 위에서 운영되는 스위치 자동 상태 점검 컨테이너
> Python SSH + Ansible EOS Collection을 두가지 자동 점검 시나리오를 구성하였으며, 환경에 맞게 선택하여 사용
> deployment(deploy.yaml) 에 사용한 이미지는 직접 docker bulid 한 이미지를 사용(ghcr.io/zpzg333/auto-check:latest) 하여 어떤 간편하게 시스템 종속성 문제 해결
> 파이썬 자동 점검 구현을 ansible을 참고하여 inventory (점검 대상) / playbook (수행 명령어 모음) / Export (jira 기반의 문서 출력) 부분으로 구분 구현
> 가변이 필요한 파일들 (ex inventory , playbook) 을 configmap 으로 구분하여 실행중인 컨테이너의 파일을 손쉽게 git에서 변경해도 argocd를 통해서 즉각 수정되도록 구

---

## docker
고객사에 완성된 ansible 파일로 자동화를 시도하였으나, ansible 버전 차이로 실행되지 않아 
docker build 수행

# 1. 베이스 이미지 : 실행될 OS 를 선정
FROM python:3.12-slim

# 2. 작업 디렉토리 : FROM python:3.9-slim OS가 실행되면 폴더생성
WORKDIR /ansible

# 3. 필수 패키지 설치 : Docker 빌드시 필요한 패키지를 설치함
RUN apt-get update && apt-get install -y --no-install-recommends \
    sshpass \
    git \
    && rm -rf /var/lib/apt/lists/* && \ 
    pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir \
    ansible-core==2.16.0 \
    paramiko \
    ansible-pylibssh && \  
    ansible-galaxy collection install arista.eos

ENTRYPOINT ["/bin/bash"]

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

## ArgoCD 연동

이 프로젝트는 **ArgoCD(GitOps)** 와 연동되어 있습니다.  
GitHub 레포지토리의 YAML 파일을 수정하면 ArgoCD가 자동으로 감지하여 Kubernetes 클러스터에 배포합니다.  
`kubectl apply` 없이 **git push만으로 배포가 완료**됩니다.

```
GitHub push (YAML 수정)
        ↓
ArgoCD 자동 감지 (auto-sync)
        ↓
Kubernetes 자동 배포 (Deployment / ConfigMap 반영)
```

| 항목 | 내용 |
|---|---|
| GitOps 도구 | ArgoCD |
| 배포 트리거 | GitHub push |
| 대상 | deploy.yaml, svc.yaml, ConfigMap 전체 |
| ArgoCD 서비스 | www.syargocd.com |

## 배경

Arista 스위치 장비 점검 업무를 자동화하여 반복 작업을 줄이고,  
이상 징후를 조기에 탐지할 수 있도록 구성한 실무 프로젝트입니다.
