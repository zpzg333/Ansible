# Arista Network Auto-Check System

* Kubernetes 위에서 운영되는 스위치 자동 상태 점검 컨테이너
* SSH 기반 Python + Ansible 두가지 자동 점검 시나리오를 구성
* deployment(deploy.yaml) 에 사용한 이미지는 직접 docker bulid 한 이미지를 사용(ghcr.io/zpzg333/auto-check:latest) 시스템 종속성 문제 해결
* 파이썬 자동 점검 구현을 ansible을 참고하여 inventory = devicelist / playbook = command 부분으로 구분 구현
* 가변이 필요한 파일을 configmap 으로 구성하여 실행중인 컨테이너의 파일을 손쉽게 git에서 변경해도 argocd를 통해서 즉각 수정되도록 구현

---

## docker iamge
고객사에 완성된 ansible 파일로 자동화를 시도하였으나, ansible 버전 차이로 실행되지 않아 
docker build 수행

1. 베이스 이미지 : 실행될 OS 를 선정
FROM python:3.12-slim

2. 작업 디렉토리 : FROM python:3.9-slim OS가 실행되면 폴더생성
WORKDIR /ansible

3. 필수 패키지 설치 : Docker 빌드시 필요한 패키지를 설치함
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
