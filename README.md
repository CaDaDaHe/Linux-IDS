# 🛡️ Linux-IDS

## 🎯 프로젝트 목표

- 리눅스 시스템에서 효율적인 **파일 검색 및 로그 분석** 기법 학습  
- `grep`, `find`, `cron`, `uptime` 등을 활용한 **자동화된 시스템 모니터링** 실습  
- IT 인프라 관점에서 **로그 기록 및 추후 활용 가능한 데이터 관리**  
- 스크립트 기반으로 반복적인 작업을 자동화하여 **운영 효율성 향상**  
- 실전 경험을 쌓기 위한 **작업 예약(Crontab) 및 로그 저장** 연습 

<br>

## ⚙️ 프로젝트 개요
본 프로젝트는 MobaXterm을 통해 하나의 서버에 다수의 사용자가 세션으로 접속하는 환경에서,  
**SSH 로그인 실패 기록을 기반으로 블랙리스트를 자동 관리하는 시스템**을 구축하는 것을 목표로 합니다.<br>

로그인 시도 데이터를 수집하고, 일정 횟수 이상 로그인 실패한 사용자를 블랙리스트에 등록하여 보안성을 향상시키는 자동화된 침입 탐지 시스템(IDS)을 구현합니다.

<br>

## 🔍 주요 기능
- SSH 로그인 실패 로그 자동 수집 (`/var/log/auth.log` 분석)
- 블랙리스트 시스템 구축 (3회 이상 로그인 실패 시 자동 등록)
- Crontab을 이용한 주기적 모니터링 및 블랙리스트 갱신
- 보안 강화 및 침입 탐지를 위한 로그 관리

<br>

## 📝 실행 과정
### **1️⃣ 5분마다 SSH 로그인 실패 로그 저장**
- `/var/log/auth.log`에서 SSH 로그인 실패 로그를 추출하여 `fail.log`에 저장
- 5분마다 자동 실행되도록 `crontab` 등록

<br>

#### **📌 스크립트 생성 및 설정**
```
sudo nano /usr/local/bin/ssh_fail_log.sh

#!/bin/bash

# 로그 파일 설정
LOG_FILE="/var/log/auth.log"

# 로그 저장 경로 설정
BASE_DIR="/var/log/ssh_fail_logs"
TODAY=$(date "+%Y-%m-%d")
LOG_DIR="$BASE_DIR/$TODAY"

# 저장할 파일
FAIL_LOG="$LOG_DIR/fail.log"

# 디렉토리 생성 (날짜별 관리)
mkdir -p "$LOG_DIR"

# SSH 로그인 실패 로그 저장
sudo grep "sshd.*Failed" "$LOG_FILE" >> "$FAIL_LOG"

exit 0
```

#### **📌 실행 권한 부여**
```
sudo chmod +x /usr/local/bin/ssh_fail_log.sh
```

<br>

### **2️⃣ 하루에 한 번 블랙리스트 갱신 및 차단**
- 하루 동안 `fail.log`를 분석하여 3번 이상 로그인 실패한 IP를 추출
- 블랙리스트(`blacklist.txt`)에 저장 후 iptables을 사용해 자동 차단

<br>

#### **📌 스크립트 생성 및 설정**
```
sudo nano /usr/local/bin/ssh_blacklist_update.sh

#!/bin/bash

# 로그 파일 설정
BASE_DIR="/var/log/ssh_fail_logs"
TODAY=$(date "+%Y-%m-%d")
LOG_DIR="$BASE_DIR/$TODAY"

# 저장할 파일
FAIL_LOG="$LOG_DIR/fail.log"
BLACKLIST_FILE="$LOG_DIR/blacklist.txt"

# 3회 이상 로그인 실패한 IP 추출
awk '{print $11}' "$FAIL_LOG" | sort | uniq -c | awk '$1 >= 3 {print $2}' > "$BLACKLIST_FILE"

# 블랙리스트 IP 차단
while read -r ip; do
    if [ ! -z "$ip" ]; then
        sudo iptables -A INPUT -s "$ip" -j DROP
        echo "$(date '+%Y-%m-%d %H:%M:%S') - Blocked $ip" >> "$LOG_DIR/blocked_ips.log"
    fi
done < "$BLACKLIST_FILE"

exit 0
```
<br>

#### **📌 실행 권한 부여**
```
sudo chmod +x /usr/local/bin/ssh_blacklist_update.sh
```

<br>

### **3️⃣ Crontab 설정**
- Crontab 편집 모드 실행
```
crontab -e
```

<br>

- 5분마다 SSH 로그인 실패 로그 저장 (ssh_fail_log.sh)
```
*/5 * * * * /bin/bash /usr/local/bin/ssh_fail_log.sh
```

<br>

- 하루에 한 번 블랙리스트 갱신 (ssh_blacklist_update.sh)
```
0 0 * * * /bin/bash /usr/local/bin/ssh_blacklist_update.sh
```
<br>

## 🚨 트러블슈팅 : 크론에서 `sudo` 명령어 실행 문제 해결  

### 문제 : 크론 작업에서 `sudo` 명령어가 정상적으로 실행되지 않음  
- 크론 작업은 기본적으로 일반 사용자 권한으로 실행되므로 `sudo`가 필요할 경우 실행되지 않을 수 있음  
- `/var/log/auth.log` 파일이 루트 전용 접근 권한을 가지고 있어 읽을 수 없을 수도 있음  

<br>

### ✅ 해결 방법  

### 1️⃣ 크론을 루트 사용자로 실행  
- 크론을 루트 권한으로 설정하여 실행하면 `sudo` 없이도 실행 가능  
```
sudo crontab -e
```

#### 📌 아래와 같이 크론 작업을 추가
```
*/5 * * * * /bin/bash /usr/local/bin/ssh_fail_log.sh
```

<br>

### 2️⃣ sudo 없이 실행하도록 스크립트 수정
- ssh_fail_log.sh 스크립트에서 sudo 제거 후, 크론 작업을 루트 권한으로 실행
```
#!/bin/bash

# 로그 파일 설정
LOG_FILE="/var/log/auth.log"

# 로그 저장 경로 설정
BASE_DIR="/var/log/ssh_fail_logs"
TODAY=$(date "+%Y-%m-%d")
LOG_DIR="$BASE_DIR/$TODAY"

# 저장할 파일
FAIL_LOG="$LOG_DIR/fail.log"

# 디렉토리 생성 (날짜별 관리)
mkdir -p "$LOG_DIR"

# 5분마다 실패 로그 저장
cat "$LOG_FILE" | grep "sshd.*" | grep "Failed" > "$FAIL_LOG"

exit 0
```

<br>

#### 📌 이후 루트 사용자로 크론 등록
```
sudo crontab -e

*/5 * * * * /bin/bash /usr/local/bin/ssh_fail_log.sh
```

<br>

### 3️⃣ auth.log 접근 권한 문제 해결
- 크론이 실행될 때 auth.log에 접근할 수 없는 경우, 권한을 조정
```
sudo chmod 644 /var/log/auth.log
```
