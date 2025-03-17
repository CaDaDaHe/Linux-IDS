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

![image](https://github.com/user-attachments/assets/df2fe429-d384-4120-8252-f62a9890caa9)


![image](https://github.com/user-attachments/assets/52139ce3-6f0c-4f78-9cd8-d649d16f3dc5)


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
sudo cat "$LOG_FILE" | grep "sshd.*" | grep "Failed" > "$FAIL_LOG"

exit 0
```

#### **📌 실행 권한 부여**
```
sudo chmod +x /usr/local/bin/ssh_fail_log.sh
```

<br>

### **2️⃣ 하루에 한 번 블랙리스트 갱신 및 차단**
- 하루 동안 `fail.log`를 분석하여 3번 이상 로그인 실패한 IP를 추출
- 블랙리스트(`blacklist.txt`)에 저장 후 iptables을 사용해 자동 차단 (추후 개선)

![image](https://github.com/user-attachments/assets/15b7a83f-5047-442f-86d4-9805e7537213)

![image](https://github.com/user-attachments/assets/e4244827-75b9-40fe-8b4c-6b9bac21cd20)


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

#날짜 구분없이 하나의 파일로 관리
BLACKLIST_FILE="$BASE_DIR/blacklist.txt"

# 3회 이상 로그인 실패한 IP 추출
awk '{print $11}' "$FAIL_LOG" | sort | uniq -c | awk '$1 >= 3 {print $2}' > "$BLACKLIST_FILE"

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

## 🚨 트러블슈팅 : 크론이 실행되지 않음
![image](https://github.com/user-attachments/assets/385b9d42-c0aa-4eb8-b43d-2491b25d3c34)


### 문제 파악: 로그인 실패기록을 저장한 auth.log 파일에서 adm 사용자는 read 권한을 가지고 있음

![image](https://github.com/user-attachments/assets/ab0a78f8-0fc5-4107-a2e6-47156b44048d)


### 문제 : 크론을 실행하는 ubuntu 사용자와 스크립트 안의 sudo가 사용되 권한 차이로 실행되지 않음

- 크론 ubuntu 사용자 권한으로 설정하였지만 스크립트 파일의 명령을 sudo로 사용하여 권한 차이로 크론이 정상적으로 실행되지 않음
- ubuntu가 sh파일의 실행권한이 없어서 실행하지 못한 것을 확인

<br>

### ✅ 해결 방법

### 1️⃣ 크론을 루트 사용자로 실행

- 크론을 루트 권한으로 설정하여 실행하여 파일에 권한 부여없이 실행 가능하게 했음

```
sudo crontab -e

```

<br>

### 2️⃣ 스크립트에 사용되던 sudo 명령을 제거

```
crontab -e

```

#### 수정 전

```
# SSH 로그인 실패 로그 저장
sudo cat "$LOG_FILE" | grep "sshd.*" | grep "Failed" > "$FAIL_LOG"

```

#### 수정 후

```
# SSH 로그인 실패 로그 저장
cat "$LOG_FILE" | grep "sshd.*" | grep "Failed" > "$FAIL_LOG"

```

<br>


## 🚨 트러블슈팅 : ip 이외에도 출력되는 문제

![image](https://github.com/user-attachments/assets/07938c2f-4bfe-4d0c-955d-92746dc903c1)


## 해결 : 정규식을 표현애 ip와 같은 형식만 출력하도록 수정
```
# SSH 로그인 실패 로그에서 IP 주소 추출하여 저장
grep "Failed password" "$LOG_FILE" | grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' > "$FAIL_LOG"
```


