# ProxMox IaC 프로젝트 사용법

## 개요
이 프로젝트는 ProxMox VE 서버의 초기 설정을 자동화하기 위한 Ansible 기반 IaC 코드입니다.

## 사전 요구사항
1. ProxMox VE가 설치된 서버
2. 서버에 SSH로 패스워드 인증 접근이 가능한 상태
3. 로컬 시스템에 Ansible 및 sshpass 설치:
   ```bash
   # macOS
   brew install ansible sshpass
   
   # 또는 pip로 Ansible만 설치하고 sshpass는 별도 설치
   pip install ansible
   brew install sshpass  # macOS
   
   # Ubuntu/Debian
   # sudo apt update && sudo apt install ansible sshpass
   
   # CentOS/RHEL
   # sudo yum install ansible sshpass
   ```

## 설정 변경이 필요한 파일들

### ansible/inventory/hosts.yml (유일한 설정 파일)
**반드시 수정해야 하는 부분:**
```yaml
ansible_host: 192.168.0.100           # ← 실제 ProxMox 서버의 IP 주소로 변경
ansible_ssh_pass: "your-password"     # ← 실제 root 패스워드로 변경
```

**선택적으로 수정할 수 있는 부분:**
```yaml
ansible_port: 22                      # ← SSH 포트가 다르면 변경
```

### SSH 키 경로
플레이북에서 자동으로 다음 키들을 찾아 사용합니다:
- `~/.ssh/id_ed25519.pub` (ed25519 키, 권장)
- `~/.ssh/id_rsa.pub` (RSA 키, 호환성)

SSH 키가 다른 경로에 있다면 플레이북의 SSH 키 배포 부분을 수정해야 합니다.

## 사용 방법

### Ansible을 이용한 ProxMox 서버 설정

#### 1. SSH 키 생성 (필요한 경우)
```bash
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
```

#### 2. 인벤토리 파일 설정
`inventory/hosts.yml` 파일에서 ProxMox 서버 정보를 수정하세요:
```yaml
proxmox_servers:
  hosts:
    proxmox-01:
      ansible_host: YOUR_PROXMOX_IP
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
      proxmox_node_name: YOUR_NODE_NAME
```

#### 3. 전체 설정 실행
```bash
ansible-playbook -i inventory/hosts.yml playbooks/00-setup-all.yml
```

또는 개별 플레이북 실행:
```bash
# SSH 키 설정
ansible-playbook -i inventory/hosts.yml playbooks/01-setup-ssh-keys.yml

# APT 저장소 설정  
ansible-playbook -i inventory/hosts.yml playbooks/02-setup-proxmox-apt.yml

# 개발 도구 설치
ansible-playbook -i inventory/hosts.yml playbooks/03-install-devtools.yml

# Docker Compose 설치
ansible-playbook -i inventory/hosts.yml playbooks/04-install-docker-compose.yml

# Bash 설정
ansible-playbook -i inventory/hosts.yml playbooks/05-setup-bash-config.yml

# Fail2ban 설정
ansible-playbook -i inventory/hosts.yml playbooks/06-setup-fail2ban.yml
```

### 도커용 VM 생성 및 설정

Docker Compose를 실행할 VM은 수동으로 생성하면, 설정은 자동으로 할 수 있습니다.

---

### 1단계: 설정 파일 준비

```bash
# 인벤토리 파일 복사 (최초 1회만)
cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml

# 서버 정보 입력
vi ansible/inventory/hosts.yml
```

### 2단계: SSH 키 기반 인증 설정 (필수 - 첫 번째)

```bash
cd ansible
ansible-playbook playbooks/01-setup-ssh-keys.yml
```

이 플레이북은 다음 단계를 수행합니다:
1. **설정 전 테스트**: 현재 인증 방법 확인
2. **SSH 키 배포**: 공개키를 서버에 배포
3. **중간 검증**: SSH 키 인증이 작동하는지 확인
4. **보안 강화**: 패스워드 인증 비활성화
5. **최종 검증**: SSH 키는 성공, 패스워드는 차단되는지 확인

### 3단계: 연결 테스트 및 확인

**Ansible로 테스트:**
```bash
# 기본 연결 테스트
ansible proxmox-server -m ping

# SSH 키만 사용하여 테스트
ansible proxmox-server -m ping --ssh-extra-args="-o PreferredAuthentications=publickey"

# 패스워드 연결 시도 (실패해야 정상)
ansible proxmox-server -m ping --ssh-extra-args="-o PreferredAuthentications=password"
```

**직접 SSH 연결:**
```bash
# SSH 키로 연결 (성공해야 함)
ssh root@192.168.0.100

# 패스워드로 연결 시도 (차단되어야 함)
ssh -o PreferredAuthentications=password root@192.168.0.100
# → "Permission denied" 메시지가 나와야 정상
```

### 4단계: ProxMox APT 저장소 설정 (권장 - 두 번째)

⚠️ **중요**: ProxMox VE는 기본적으로 유료 Enterprise 저장소를 사용하도록 설정되어 있어, 
구독 없이는 `apt update` 시 401 Unauthorized 에러가 발생합니다.

**APT 저장소 자동 설정:**
```bash
cd ansible
ansible-playbook playbooks/02-setup-proxmox-apt.yml
```

이 플레이북은 다음을 수행합니다:
1. **현재 상태 확인**: 저장소 설정 및 에러 상태 분석
2. **Enterprise 저장소 비활성화**: 401 Unauthorized 에러 해결
3. **No-Subscription 저장소 활성화**: 무료 저장소 추가
4. **APT 업데이트**: 패키지 목록 자동 업데이트
5. **설정 검증**: 모든 설정이 올바른지 확인

**APT 저장소 설정 후 사용 가능한 명령:**
```bash
# ProxMox 서버에서 직접 실행 가능
ssh root@192.168.0.100

# 패키지 목록 업데이트
apt update

# 시스템 업그레이드
apt upgrade

# 패키지 설치
apt install <패키지명>
```

## 재실행 시 주의사항

### 순서대로 실행하기

```bash
# 1️⃣ SSH 키 설정 (필수 - 가장 먼저)
ansible-playbook playbooks/01-setup-ssh-keys.yml

# 2️⃣ APT 저장소 설정 (권장 - SSH 설정 후)
ansible-playbook playbooks/02-setup-proxmox-apt.yml
```

### 개별 플레이북 재실행

SSH 키 설정이 완료된 후에는 다음과 같이 실행해야 합니다:

```bash
# SSH 키 설정 완료 후 재실행 (패스워드 불필요)
ansible-playbook playbooks/01-setup-ssh-keys.yml

# 특정 태그만 실행
ansible-playbook playbooks/01-setup-ssh-keys.yml --tags ssh_config

# APT 저장소만 다시 설정
ansible-playbook playbooks/02-setup-proxmox-apt.yml
```

### 특정 단계부터 다시 시작하기

플레이북 실행 중 오류가 발생하거나 특정 작업부터 다시 실행하고 싶은 경우:

#### `--start-at-task` 옵션 사용

```bash
# 전체 설정에서 특정 단계부터 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/00-setup-all.yml \
  --start-at-task="3단계: 개발 도구 설치"

# Docker 설치에서 특정 단계부터 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06-install-docker-compose.yml \
  --start-at-task="14단계: Docker 그룹에 사용자 추가 (선택사항)"

# SSH 키 설정에서 특정 단계부터 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/01-setup-ssh-keys.yml \
  --start-at-task="4단계: SSH 보안 설정 적용"

# Bash 설정에서 특정 단계부터 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/04-setup-bash-config.yml \
  --start-at-task="5단계: 사용자별 Bash 설정 적용"
```

#### 일반적인 재시작 시나리오

```bash
# Docker 설치 실패 후 그룹 추가부터 다시 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06-install-docker-compose.yml \
  --start-at-task="14단계: Docker 그룹에 사용자 추가 (선택사항)"

# APT 저장소 설정 실패 후 저장소 추가부터 다시 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/02-setup-proxmox-apt.yml \
  --start-at-task="3단계: No-Subscription 저장소 추가"

# SSH 키 설정에서 보안 설정부터 다시 시작
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/01-setup-ssh-keys.yml \
  --start-at-task="4단계: SSH 보안 설정 적용"
```

#### 태스크 이름 확인 방법

플레이북의 태스크 이름을 확인하려면:

```bash
# 플레이북의 모든 태스크 목록 확인
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06-install-docker-compose.yml --list-tasks

# 특정 플레이북의 태스크 이름들 확인
grep -n "name:" ansible/playbooks/06-install-docker-compose.yml
```

#### Dry-run으로 확인

실제 실행 전에 어떤 작업이 수행될지 확인:

```bash
# 변경사항만 확인 (실제로 실행하지 않음)
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06-install-docker-compose.yml \
  --start-at-task="14단계: Docker 그룹에 사용자 추가 (선택사항)" --check

# 상세한 출력으로 확인
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/06-install-docker-compose.yml \
  --start-at-task="14단계: Docker 그룹에 사용자 추가 (선택사항)" --check -v
```

### 플레이북 자동 감지 기능

플레이북들은 현재 연결 방식을 자동으로 감지하여:
- **SSH 키 설정 플레이북**: 
  - 패스워드 연결: SSH 키 배포 실행
  - SSH 키 연결: SSH 키 배포 건너뛰고 설정만 업데이트
- **APT 저장소 플레이북**: 
  - 이미 설정된 경우: 설정 상태만 확인하고 건너뜀
  - 미설정된 경우: 전체 설정 과정 실행

## 주의사항

⚠️ **중요**: 설정 변경 전 현재 SSH 설정을 백업하세요!

⚠️ **보안**: 초기 설정 후에는 패스워드 인증을 비활성화하므로, SSH 키를 분실하지 않도록 주의하세요.

⚠️ **테스트**: 가능하면 별도 테스트 환경에서 먼저 실행해보세요.

## 문제 해결

### sshpass 관련 오류
```
to use the 'ssh' connection type with passwords, you must install the sshpass program
```
→ `brew install sshpass` (macOS) 또는 해당 OS의 패키지 매니저로 sshpass 설치

### SSH 연결 실패
- 방화벽 설정 확인
- SSH 서비스 상태 확인
- IP 주소와 포트 번호 재확인
- SSH 멀티플렉싱 캐시 정리: `ssh -O exit -o ControlPath="/tmp/ansible-%h-%p-%r" <서버IP>`

### 인증 시도 횟수 초과 오류
```
Too many authentication failures
Received disconnect from <IP> port 22:2: Too many authentication failures
```
**ProxMox 서버에서** (웹 콘솔 또는 직접 접속):
```bash
# SSH 서비스 재시작으로 연결 제한 해제
systemctl restart sshd

# 또는 임시로 인증 시도 횟수 증가
sed -i 's/MaxAuthTries 3/MaxAuthTries 10/' /etc/ssh/sshd_config
systemctl restart sshd
```

### 패스워드 인증 실패
```
fatal: [proxmox-server]: UNREACHABLE! => changed=false 
  msg: |-
    Invalid/incorrect password: Warning: Permanently added '192.168.0.100' (ED25519) to the list of known hosts.
    Permission denied, please try again.
  unreachable: true
```
→ `ansible/inventory/hosts.yml`에서 `ansible_ssh_pass` 값 확인

### 권한 오류
- ProxMox에서 root 계정 SSH 접근이 허용되어 있는지 확인
- `/etc/ssh/sshd_config`에서 `PermitRootLogin yes` 설정 확인

### 키 인증 실패
- SSH 키가 올바르게 생성되었는지 확인
- authorized_keys 파일 권한 확인 (600)

## 프로젝트 구조

```
homeserver-iac/
├── README.md
├── inventory/
│   └── hosts.yml                    # 서버 인벤토리
└── playbooks/                       # Ansible 플레이북들
    ├── 00-setup-all.yml            # 전체 설정 실행
    ├── 01-setup-ssh-keys.yml       # SSH 키 설정
    ├── 02-setup-proxmox-apt.yml    # APT 저장소 설정
    ├── 03-install-devtools.yml     # 개발 도구 설치
    ├── 04-install-docker-compose.yml # Docker Compose 설치
    ├── 05-setup-bash-config.yml    # Bash 환경 설정
    └── 06-setup-fail2ban.yml       # Fail2ban 보안 설정
```

## 다음 단계
SSH 키 설정이 완료되면 추가 보안 설정을 진행할 수 있습니다:
- 방화벽 설정
- 사용자 계정 생성
- 시스템 업데이트
- 모니터링 설정
