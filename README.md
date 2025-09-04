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

## 사용법

### 1단계: 설정 파일 준비

```bash
# 인벤토리 파일 복사 (최초 1회만)
cp ansible/inventory/hosts.yml.example ansible/inventory/hosts.yml

# 서버 정보 입력
vi ansible/inventory/hosts.yml
```

### 2단계: SSH 키 기반 인증 설정

```bash
cd ansible
ansible-playbook playbooks/setup-ssh-keys.yml
```

이 플레이북은 다음 단계를 수행합니다:
1. **설정 전 테스트**: 현재 인증 방법 확인
2. **SSH 키 배포**: 공개키를 서버에 배포
3. **중간 검증**: SSH 키 인증이 작동하는지 확인
4. **보안 강화**: 패스워드 인증 비활성화
5. **최종 검증**: SSH 키는 성공, 패스워드는 차단되는지 확인

### 3단계: 연결 테스트

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

## 재실행 시 주의사항

SSH 키 설정이 완료된 후에는 다음과 같이 실행해야 합니다:

```bash
# SSH 키 설정 완료 후 재실행 (패스워드 불필요)
ansible-playbook playbooks/setup-ssh-keys.yml

# 특정 태그만 실행
ansible-playbook playbooks/setup-ssh-keys.yml --tags ssh_config
```

플레이북은 현재 연결 방식을 자동으로 감지하여:
- **패스워드 연결**: SSH 키 배포 실행
- **SSH 키 연결**: SSH 키 배포 건너뛰고 설정만 업데이트

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

## 다음 단계
SSH 키 설정이 완료되면 추가 보안 설정을 진행할 수 있습니다:
- 방화벽 설정
- 사용자 계정 생성
- 시스템 업데이트
- 모니터링 설정
