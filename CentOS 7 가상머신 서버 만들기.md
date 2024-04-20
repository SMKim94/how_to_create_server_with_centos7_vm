# CentOS 7 가상머신 서버 만들기

## 목차
1. [CentOS 7 설치](#centos-7-설치)
2. [Docker 설치](#docker-설치)

## CentOS 7 설치
- Oracle VM VirtualBox 홈페이지 : https://www.virtualbox.org/
- CentOS 파일 : [CentOS-7-x86_64-Everything-2009.iso](https://mirror.kakao.com/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Everything-2009.iso) 

### 파티션
- 참고 : https://jootc.com/p/201806031149

1. 설치 요약 - 시스템 - 설치 대상 선택
1. 설치 대상 - 장치 선택 - 로컬 표준 디스크 - 디스크 선택
2. 설치 대상 - 기타 저장소 옵션 - 파티션을 설정합니다. 체크
3. 설치 대상 - 왼쪽 위의 완료 버튼 선택하여 수동으로 파티션 설정 화면 이동
4. 수동으로 파티션 설정에서 파티션 설정하기
   - 추천 설정
   - | 마운트 지점 | 용량             | 장치 유형   | 파일 시스템 |
     |-------------|------------------|-------------|-------------|
     | /boot       | 500M             | 표준 파티션 | ext4        |
     | swap        | RAM 용량 2배     | 표준 파티션 | swap        |
     | /home       | 용도에 따라      | LVM         | ext4        |
     | /           | 비워두기(나머지) | LVM         | ext4        |
5. 수동으로 파티션 설정 - 상단의 완료 버튼 선택
6. 수동으로 파티션 설정 - 변경 요약 - 변경 사항 적용 선택
7. 설정 완료

### 네트워크

네트워크 방식 : 어댑터에 ***브리지***

|위치       | IP            |
|:---------:|:-------------:|
| 사용자 PC | 192.168.100.2 |
| 가상머신  | 192.168.100.5 |

#### Oracle VM VirtualBox 에서 네트워크 설정
1. 가상머신 창 위쪽의 장치 선택
2. 네트워크 선택
3. 네트워크 설정... 선택
4. 어댑터 1 - 다음에 연결됨 - 어댑터에 브리지 선택

#### CentOS 설치 시 네트워크 설정 (보충 필요)
1. 네트워크 선택
2. 이더넷 IP 설정
3. 호스트 이름 설정
4. 왼쪽 위의 완료 버튼 선택

### 계정 정보
| 아이디 | 비밀번호     | 타입          |
|:------:|:-------------|---------------|
| root   | ROQKFfnxm001 | 최고 관리자   |
| mgr    | AOSLWJ001!   | 관리자(wheel) |

### 초기 설정 명령어
```shell
# 패키지 업데이트
sudo yum update -y

# 필요한 도구 설치
sudo yum install net-tools vim -y
```

### SSH 설정
- SSH 접속을 위한 포트를 기본 22/tcp에서 2222/tcp로 변경하여 사용한다.

#### 포트 변경 - SELinux에 SSH 포트 추가
```shell
# SELinux 설정을 위한 SEManage 설치
sudo yum install policycoreutils-python -y

# SELinux에 SSH 포트를 2222/tcp 로 설정
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

#### 포트 변경 - 방화벽 설정
```shell
# 2222/tcp 포트 영구 허용
sudo firewall-cmd --permanent --add-port=2222/tcp

# 방화벽 재시작
sudo firewall-cmd --reload
```

#### 포트 변경 - SSH 설정 파일 수정
```shell
# SSH 설정 파일 열기
sudo vim /etc/ssh/sshd_config

# #Port 22를 Port 2222 로 변경
Port 2222

# SSH 설정 파일 저장
:wq
```
또는
```shell
# SSH 설정 파일에서 #Port 설정 확인
sudo grep '#Port' /etc/ssh/sshd_config

# Port 주석 제거 및 2222 포트로 변경
sudo sed -i '/^#Port 22/c\Port 2222' /etc/ssh/sshd_config

# SSH 서비스 재시작
sudo systemctl reload sshd
```

#### 키 파일 - 생성
사용자 PC에서 개인 키, 공개 키 파일 생성.
```bat
ssh-keygen -t <암호화 방식> -b <키의 길이> -f <키 파일 이름 저장> -a <KDF 반복 횟수> -C <주석>
```
아래는 Windows cmd로 실행한 결과이다.
```bat
REM 키 생성 명령어
C:\Users\<사용자 계정>\.ssh>ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.

REM 키 파일 이름 설정
Enter file in which to save the key (C:\Users\ksm/.ssh/id_rsa): <키 파일 이름>

REM 키 파일 비밀번호 설정
Enter passphrase (empty for no passphrase):

REM 키 파일 비밀번호 재입력
Enter same passphrase again:

REM 생성 결과
Your identification has been saved in <키 파일 이름>
Your public key has been saved in <키 파일 이름>.pub
The key fingerprint is:
SHA256:<암호화 텍스트> <사용자 계정>@<PC 이름>
The key's randomart image is:
+---[RSA 4096]----+
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
|                 |
+----[SHA256]-----+
```

#### 키 파일 - 가상머신으로 공개 키 파일 전송
Secure Copy Protocol(SCP)를 사용하여 사용자 PC에서 가상머신으로 공개 키 파일을 전송한다.
```bat
scp -P <포트 번호> "<파일>" <원격지 계정>@<원격지 주소>:<원격지 폴더 위치>
```
아래는 Windows cmd로 실행한 결과이다.
```bat
C:\Users\<사용자 계정>\.ssh>scp -P 2222 "C:\Users\<사용자 계정>\.ssh\<키 파일 이름>.pub" 
mgr@192.168.100.5's password: <비밀번호 입력>
<키 파일 이름>.pub          100%  748   373.3KB/s   00:00
```

#### 키 파일 - 가상머신에 공개 키 등록
- 가상머신에서 공개 키를 등록하려면 authorized_keys 파일에 공개 키의 내용을 추가한다.
- authorized_keys 파일이 없다면 파일을 생성하면 된다.
- authorized_keys 파일은 공개 키를 저장하는 매우 중요한 파일이기 때문에 다른 사용자가 파일을 읽거나 수정할 수 없도록 권한을 제한하는 것이 좋다.

```shell
# authorized_keys 파일 마지막에 공개 키 내용을 추가한다.
# authorized_keys 파일이 없다면 authorized_keys 파일을 생성한다.
cat <키 파일 이름>.pub >> authorized_keys

# authrozied_keys 파일의 권한을 제한한다.
chmod 600 authorized_keys

# 공개 키의 내용을 authorized_keys 파일에 추가했기 때문에 공개 키 파일은 제거한다.
rm <키 파일 이름>.pub
```

- 참고 : 왜 authorized_keys 파일이냐면, SSH 설정 파일(```/etc/ssh/sshd_config```)의 ***AuthorizedKeysFile*** 옵션에 설정되어 있어서 그렇다.

#### 사용자 PC에서 SSH로 가상머신 접속
```bat
ssh -i "<사용자 PC의 개인 키 파일>" -p <포트 번호> <원격지 계정>@<원격지 주소>
```
또는 PuTTY나 MobaXterm으로 접속한다.

## Docker 설치

### 설치 준비
참고 : [Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

#### (옵션) 이전 버전 Docker 삭제
만약을 위해 이전 버전의 Docker를 삭제한다.
```shell
# 한 줄
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```
```shell
# 여러 줄
sudo yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine
```

나머지 Docker 관련 파일 제거
```shell
rm -rf /var/lib/docker
```

#### yum-utils 패키지 설치
- yum 관련 유틸리티 도구.
- 저장소를 관리하는 ***yum-config-manage***이 포함됨.
- Docker의 경우 공식 CentOS 저장소가 아닌 Docker 자체의 저장소를 사용함.
- ***yum-config-manager***를 사용하여 Docker 저장소를 추가, 관리할 수 있음.
```shell
sudo yum install yum-utils -y
```

#### Docker 저장소 추가
```shell
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

#### Docker 패키지 설치
```shell
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

#### Docker 서비스 실행
```shell
sudo systemctl start docker
```
정상 실행되었다면 아래와 같이 나타난다.
```shell
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

#### Docker 서비스 자동 실행 설정
```shell
sudo systemctl enable docker
```

#### Docker 테스트
```shell
sudo docker run hello-world
```
