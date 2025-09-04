# ProxMox IaC 프로젝트 계획서

## 프로젝트 개요
ProxMox 설치 직후 SSH 키 기반 인증 설정 및 기본 보안 구성을 자동화하는 IaC 프로젝트

## 현재 상황 (가정)
- ProxMox VE 설치 완료
- SSH 서비스 활성화됨
- 패스워드 기반 SSH 로그인 가능 (root 계정)
- 네트워크 연결 정상

## 1단계: SSH 키 기반 인증 설정

### Ansible을 이용한 접근 방식

#### 디렉토리 구조
```
homeserver-iac/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── proxmox.yml
│   ├── playbooks/
│   │   ├── setup-ssh-keys.yml
│   │   └── initial-setup.yml
│   ├── roles/
│   │   └── ssh-setup/
│   │       ├── tasks/main.yml
│   │       ├── handlers/main.yml
│   │       └── templates/
│   │           └── sshd_config.j2
│   └── host_vars/
└── terraform/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── terraform.tfvars.example
```

#### Ansible 설정 세부 사항

##### 1. ansible.cfg
- 기본 설정 파일
- SSH 연결 설정
- 호스트 키 검증 비활성화 (초기 설정 시)

##### 2. inventory/hosts.yml
```yaml
all:
  children:
    proxmox:
      hosts:
        pve-01:
          ansible_host: 192.168.1.100
          ansible_user: root
          ansible_ssh_pass: "{{ vault_root_password }}"
```

##### 3. playbooks/setup-ssh-keys.yml
- SSH 공개키 배포
- authorized_keys 파일 설정
- SSH 설정 보안 강화
- 패스워드 인증 비활성화

##### 4. roles/ssh-setup/tasks/main.yml
주요 작업:
- 로컬 SSH 키 존재 확인 (없으면 생성)
- 공개키를 원격 서버에 복사
- SSH 서비스 설정 업데이트
- SSH 서비스 재시작

### Terraform을 이용한 접근 방식

#### Terraform Provider
- **telmate/proxmox** provider 사용
- ProxMox API를 통한 관리

#### 주요 리소스
1. `proxmox_vm_qemu`: VM 생성 및 관리
2. `local_file`: SSH 키 관리
3. `null_resource`: provisioner를 통한 초기 설정

#### Terraform 설정 파일들

##### variables.tf
```hcl
variable "proxmox_host" {
  description = "ProxMox 서버 IP 주소"
  type        = string
}

variable "proxmox_user" {
  description = "ProxMox 사용자명"
  type        = string
  default     = "root"
}

variable "ssh_public_key_path" {
  description = "SSH 공개키 파일 경로"
  type        = string
  default     = "~/.ssh/id_rsa.pub"
}
```

## 권장 접근 방식: Ansible 우선

### 이유:
1. **초기 설정에 특화**: Ansible은 기존 서버의 초기 구성에 매우 적합
2. **간단한 설정**: SSH 키 배포는 Ansible의 기본 모듈로 쉽게 처리 가능
3. **멱등성**: 여러 번 실행해도 안전
4. **디버깅 용이**: 각 단계별 상태 확인 가능

### 실행 단계:

#### 1단계: 환경 준비
```bash
# SSH 키 생성 (없는 경우)
ssh-keygen -t rsa -b 4096 -C "homeserver-admin"

# Ansible 설치
pip install ansible

# Ansible Vault로 패스워드 암호화
ansible-vault create group_vars/proxmox.yml
```

#### 2단계: 초기 연결 테스트
```bash
ansible proxmox -m ping -i inventory/hosts.yml --ask-pass
```

#### 3단계: SSH 키 배포
```bash
ansible-playbook -i inventory/hosts.yml playbooks/setup-ssh-keys.yml --ask-pass
```

#### 4단계: 키 기반 인증 확인
```bash
ansible proxmox -m ping -i inventory/hosts.yml
```

## 보안 고려사항

### SSH 설정 강화
- PasswordAuthentication no
- PermitRootLogin yes (초기에는 필요, 추후 일반 사용자 계정 생성 후 변경)
- Port 변경 (기본 22에서 다른 포트로)
- AllowUsers 제한
- MaxAuthTries 제한

### 방화벽 설정
- UFW 또는 iptables 설정
- 필요한 포트만 개방
- SSH 포트 변경 시 방화벽 규칙 업데이트

## 다음 단계 계획

### 2단계: 기본 보안 설정
- 방화벽 설정
- 시스템 업데이트
- 불필요한 서비스 비활성화
- 로그 설정

### 3단계: 사용자 계정 관리
- 관리자 계정 생성
- sudo 권한 설정
- root 직접 로그인 비활성화

### 4단계: 모니터링 설정
- 로그 수집
- 시스템 모니터링
- 알림 설정

## 파일 생성 우선순위

1. **ansible.cfg** - Ansible 기본 설정
2. **inventory/hosts.yml** - 대상 서버 정의
3. **group_vars/proxmox.yml** - 변수 정의
4. **playbooks/setup-ssh-keys.yml** - SSH 키 설정 플레이북
5. **roles/ssh-setup/** - SSH 설정 역할

## 주의사항

1. **백업**: 설정 변경 전 현재 설정 백업
2. **테스트**: 별도 환경에서 먼저 테스트
3. **롤백 계획**: 문제 발생 시 되돌릴 수 있는 방법 준비
4. **문서화**: 모든 변경 사항 문서화

## 예상 소요 시간
- 초기 환경 설정: 30분
- Ansible 코드 작성: 1-2시간
- 테스트 및 디버깅: 30분-1시간
- **총 소요 시간: 2-3.5시간**
