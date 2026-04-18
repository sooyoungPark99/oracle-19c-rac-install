# grid 설치

### 1. 설치 파일 업로드 및 압축 해제(rac1, oracle)

<aside>

아래 링크에서 Oracle Grid, Oracle Database 19c를 다운로드하여 리눅스에 업로드한다.
[https://www.oracle.com/kr/database/technologies/oracle19c-linux-downloads.html](https://www.oracle.com/kr/database/technologies/oracle19c-linux-downloads.html)

</aside>

**1번 노드의 oracle 유저로 MobaXterm에 접속해서 위의 2개 파일을 업로드한다.**

```bash
[root@rac1 ~]# su - oracle
[oracle@rac1 ~]$ ls
LINUX.X64_193000_db_home.zip  LINUX.X64_193000_grid_home.zip

[oracle@rac1 ~]$ unzip LINUX.X64_193000_grid_home.zip -d $GRID_HOME
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image.png)

---

### 2. cvuqdisk 패키지 설치(rac1/rac2, root)

<aside>

**cvuqdisk**는 Oracle RAC 설치 전 공유 디스크를 검증하기 위해 필요한 패키지이다.

1번 노드에서 설치 후 2번 노드로 전송하여 설치한다.

</aside>

```bash
# 1번 노드에서 root 유저로 설치
[root@rac1 ~]# cd $GRID_HOME/cv/rpm
[root@rac1 rpm]# rpm -ivh cvuqdisk-1.0.10-1.rpm

# 2번 노드로 rpm 전송
[root@rac1 rpm]# scp cvuqdisk-*.rpm root@192.168.56.102:/tmp
[root@rac1 rpm]# ssh root@rac2-priv
```

![  **•   yes → password 입력**](grid%20%EC%84%A4%EC%B9%98/image%201.png)

  **•   yes → password 입력**

```bash
# cvuqdisk 패키지 설치 (2번 노드)
[root@rac2 ~]# cd /tmp
[root@rac2 tmp]# rpm -ivh cvuqdisk-*.rpm
[root@rac2 tmp]# exit
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%202.png)

---

### 3. 방화벽 비활성화(rac1/rac2, root)

```bash
# rac1, rac2 양쪽 모두 수행
systemctl stop firewalld
systemctl disable firewalld
```

<aside>
📝

RAC 설치 및 클러스터 통신(공용/프라이빗/SCAN 등)에 필요한 포트가 방화벽에 의해 차단되면 설치/검증 단계에서 오류가 발생할 수 있어, 실습 환경에서는 방화벽을 일시적으로 비활성화한다.

</aside>

---

### 4. Public 네트워크를 NAT Network로 변경 + 통신 확인

#### VirtualBox 설정 변경

- rac1, rac2 VM을 둘 다 끈다.
- VirtualBox 설정에서 **네트워크 어댑터 1**을 NAT Network로 변경한다. (rac1/rac2 모두 적용)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%203.png)

![스크린샷 2026-04-08 114916.png](grid%20%EC%84%A4%EC%B9%98/%EC%8A%A4%ED%81%AC%EB%A6%B0%EC%83%B7_2026-04-08_114916.png)

#### 변경 후 통신 확인

**rac1에서 통신 확인**

```bash
[root@rac1 ~]# ping -c 3 rac1
[root@rac1 ~]# ping -c 3 rac2
[root@rac1 ~]# ping -c 3 rac1-priv
[root@rac1 ~]# ping -c 3 rac2-priv
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%204.png)

**rac2에서 통신 확인**

```bash
[root@rac2 ~]# ping -c 3 rac1
[root@rac2 ~]# ping -c 3 rac2
[root@rac2 ~]# ping -c 3 rac1-priv
[root@rac2 ~]# ping -c 3 rac2-priv
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%205.png)

---

### 5. oracle 사용자 무비밀번호 SSH 설정(rac1, oracle)

```bash
[root@rac1 ~]# su - oracle
[oracle@rac1 ~]$ cd $GRID_HOME/oui/prov/resources/scripts
[oracle@rac1 scripts]$ ./sshUserSetup.sh -user oracle -hosts "rac1 rac2" -noPromptPassphrase -advanced
```

- 값 입력 창이 나오면 `yes` 입력
- 이후 **rac1 oracle 패스워드 → rac2 oracle 패스워드** 순으로 입력(총 2번)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%206.png)

#### oracle SSH 확인(양방향)

```bash
# rac1 에서 확인
[oracle@rac1 ~]$ ssh oracle@rac1
[oracle@rac1 ~]$ ssh oracle@rac2
[oracle@rac1 ~]$ ssh oracle@rac1-priv
[oracle@rac1 ~]$ ssh oracle@rac2-priv
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%207.png)

```bash
# rac2 에서도 확인
[oracle@rac2 ~]$ ssh oracle@rac1
[oracle@rac2 ~]$ ssh oracle@rac1-priv
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%208.png)

#### root SSH 무비밀번호 설정(양방향)

```bash
# rac1에서 root 유저로 수행
[root@rac1 ~]# ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
[root@rac1 ~]# ssh-copy-id root@rac2

# rac2에서 root 유저로 수행
[root@rac2 ~]# ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
[root@rac2 ~]# ssh-copy-id root@rac1

# 양방향 확인
[root@rac1 ~]# ssh rac2 hostname
[root@rac2 ~]# ssh rac1 hostname
```

---

### 6. ASM 공유 디스크 인식 확인(양쪽 노드)

```bash
[root@rac1 ~]# oracleasm listdisks
CRS1
CRS2
CRS3
DATA1
FRA1
```

#### 디스크가 안 보일 때(조치)

<aside>

**중요**: 공유 디스크는 rac1/rac2 양쪽 모두에서 ASM 레이블이 보여야 한다.

- 레이블 생성(createdisk): **rac1에서 1회만**
- 디스크 인식(scandisks): **rac2에서**
- 파티션 생성(fdisk): **rac1, rac2 모두**

</aside>

(1) rac1: 파티션 생성 + ASM 레이블 생성

```bash
# 파티션 생성 (rac1)
for disk in sdb sdc sdd sde sdf; do
	echo -e "n\np\n1\n\n\n\nw" | fdisk /dev/$disk

done
partprobe

# ASM 레이블링 (rac1에서만 수행)
oracleasm createdisk CRS1 /dev/sdb1
oracleasm createdisk CRS2 /dev/sdc1
oracleasm createdisk CRS3 /dev/sdd1
oracleasm createdisk DATA1 /dev/sde1
oracleasm createdisk FRA1 /dev/sdf1

oracleasm listdisks
```

![**→ 이렇게 공유디스크가 잘 보이면 성공!**](grid%20%EC%84%A4%EC%B9%98/image%209.png)

**→ 이렇게 공유디스크가 잘 보이면 성공!**

(2) rac2: 파티션 생성 + ASM 디스크 인식

```bash
# 파티션 생성 (rac2)
for disk in sdb sdc sdd sde sdf; do
	echo -e "n\np\n1\n\n\n\nw" | fdisk /dev/$disk

done
partprobe

oracleasm scandisks
oracleasm listdisks
```

![**→ 공유디스크가 잘 보이면 성공!**](grid%20%EC%84%A4%EC%B9%98/image%2010.png)

**→ 공유디스크가 잘 보이면 성공!**

---

### 7. SCAN DNS 확인(rac1/rac2)

- rac1/rac2에서 모두 확인(3개 IP가 나와야 함)

```bash
[root@rac1 ~]# getent hosts rac-scan
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2011.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2012.png)

---

### 8. OEL 8.9 설치 호환(CV_ASSUME_DISTID) 설정(rac1/rac2, oracle)

- `/etc/os-release`가 8.9이면 `CV_ASSUME_DISTID` 설정이 필요하다.

```bash
[root@rac1 ~]# cat /etc/os-release
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2013.png)

<aside>

`CV_ASSUME_DISTID`는 설치 프로그램이 OS 버전 검증을 통과하도록 인식시키는 환경 변수이다.

- 지원 목록: ✅ OL 8.4/8.5/8.6, ❌ OL 8.9
- 따라서 `CV_ASSUME_DISTID=OEL8.4`로 설정해 검증을 통과시킨다.
</aside>

#### rac1 설정

```bash
[root@rac1 ~]# su - oracle
[oracle@rac1 ~]$ vi .bash_profile

# 아래 두 줄 추가
export LANG=en_US.UTF-8
export CV_ASSUME_DISTID=OEL8.4
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2014.png)

```bash
[oracle@rac1 ~]$ source .bash_profile
[oracle@rac1 ~]$ echo $CV_ASSUME_DISTID
OEL8.4
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2015.png)

#### rac2도 동일 설정

```bash
[root@rac2 ~]# su - oracle
[oracle@rac2 ~]$ vi .bash_profile

# 아래 두 줄 추가
export LANG=en_US.UTF-8
export CV_ASSUME_DISTID=OEL8.4

[oracle@rac2 ~]$ source .bash_profile
[oracle@rac2 ~]$ echo $CV_ASSUME_DISTID
OEL8.4
```

![image.png](grid%20%EC%84%A4%EC%B9%98/82e511f0-63af-436b-8b2b-aa36275e9c95.png)

---

### 9. Grid Infrastructure 설치 실행(rac1, oracle)

```bash
[oracle@rac1 ~]$ cd
[oracle@rac1 ~]$ source .bash_profile
[oracle@rac1 ~]$ cd $GRID_HOME
[oracle@rac1 19c]$ ./gridSetup.sh
```

![**설치 마법사가 실행이 안되면 껐다가 다시 oracle 유저로 접속 진행합니다.**](grid%20%EC%84%A4%EC%B9%98/image%2016.png)

**설치 마법사가 실행이 안되면 껐다가 다시 oracle 유저로 접속 진행합니다.**

#### 설치 마법사 진행

아래 이미지대로 설치를 진행한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2017.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2018.png)

- Cluster Name: racdb
- Scan Name: rac-scan
- Scan Port: 1521

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2019.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2011.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2020.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/0402b1dd-91be-42b7-a915-72320a7c54fe.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2021.png)

![**OS Password : oracle**](grid%20%EC%84%A4%EC%B9%98/image%2022.png)

**OS Password : oracle**

---

### 10. (트러블슈팅) scp 관련 에러가 날 때

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2023.png)

#### rac1,2 모두 실행

```bash
[root@rac1 ~]# cp /usr/bin/scp /usr/bin/scp.orig

# 새로운 /usr/bin/scp 파일 생성
[root@rac1 ~]# vi /usr/bin/scp

# scp 파일 내에 아래의 내용 추가
/usr/bin/scp.orig -T $*

# scp 파일의 권한 조정
[root@rac1 ~]# chmod 555 /usr/bin/scp
```

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2024.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2025.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2026.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2027.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2028.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2029.png)

디스크 그룹 이름을 기존에 적혀잇는 DATA를 지우고 CRS로 적습니다. Normal을 선택한다.

만약 경로가 나오지 않으면 Change Discovery Path를 누르고 /dev/oracleasm/disks 를 입력한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2030.png)

CRS1,2,3을 선택하고 다음으로 넘어간다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2031.png)

SYSASM의 비밀번호를 oracle로 입력한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2032.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2033.png)

Do not use IPMI를 선택한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2034.png)

EM을 체크하지 않고 다음으로 넘어간다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2035.png)

그룹을 dba로 설정한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2036.png)

Yes 선택한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2037.png)

Oracle Base는 /u01/app/oracle 경로로 설정합니다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2038.png)

orainventory도 그대로 두면 됩니다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2039.png)

클러스터 구성 과정 중 root 계정으로 스크립트를 실행하는데, 자동으로 구성 스크립트를 실행하기 위해 root 계정의 비밀번호(oracle) 를 입력해준다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2040.png)

설치 전 필요조건을 검사한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2041.png)

DNS 에러는 SCAN IP가 DNS에 등록되지 않은것이라 무시하고 진행한다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2042.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2043.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2044.png)

중간에 뜨는 에러메세지는 모두 yes를 누르고 설치를 진행합니다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2045.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2046.png)

---

### 11. 설치 완료 처리 및 설치 결과 확인

- 설치가 완료되었다면 **Skip 버튼**을 누른다.

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2047.png)

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2048.png)

- 아래 화면이 뜨면 Grid 설치 완료

![image.png](grid%20%EC%84%A4%EC%B9%98/image%2049.png)

#### 설치 결과 확인

```bash
[root@rac1 ~]# crsctl stat res -t
```

#### **scp 파일의 권한 조정 (rac1,2 모두 진행)**

```bash
[root@psy1 ~]# mv /usr/bin/scp.orig /usr/bin/scp
mv: overwrite '/usr/bin/scp'? yes
[root@psy1 ~]# chmod 755 /usr/bin/scp

# 복원 후 반드시 확인
file /usr/bin/scp
# ELF 64-bit 로 나와야 정상 (스크립트면 아직 복원 안 된 것)
```

![image (20).png](grid%20%EC%84%A4%EC%B9%98/image_(20).png)