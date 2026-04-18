# 그리드 설치전 리눅스 환경설정

- 개요

---

<aside>
<img src="https://www.notion.so/icons/push-pin_gray.svg" alt="https://www.notion.so/icons/push-pin_gray.svg" width="40px" />

그리드 설치 전 리눅스 환경 설정 절차 정리

- Grid 설치(ASM 포함)를 진행하기 전에 필요한 OS 사용자/그룹, 디렉터리, 패키지(ASMLib), 커널 파라미터, 환경변수(.bash_profile)를 설정한다.
</aside>

> **대상 : rac1 (필요 시 rac2도 동일 적용)**
> 

---

### 1. 그룹 및 패스워드 설정

<aside>

oracle 사용자를 dba 그룹의 멤버로 추가한다. (oracle 사용자는 preinstall 단계에서 생성됨)

</aside>

```bash
[root@rac1 ~]# usermod -g dba -G dba oracle
[root@rac1 ~]# passwd oracle
oracle 사용자의 비밀 번호 변경 중
새  암호:
잘못된 암호: 암호는 8 개의 문자 보다 짧습니다
새  암호 재입력:
passwd: 모든 인증 토큰이 성공적으로 업데이트 되었습니다.
[root@rac1 ~]# id oracle
uid=54321(oracle) gid=54322(dba) groups=54322(dba)
[root@rac1 ~]# cat /etc/passwd | grep oracle
oracle:x:54321:54322::/home/oracle:/bin/bash
```

---

### 2. 오라클/그리드 설치용 디렉터리 생성

- 목적: ORACLE_HOME / GRID_HOME, 인벤토리, 아카이브 저장 경로 등을 미리 생성

```bash
mkdir -p /home/oracle/media
mkdir -p /u01/app/oracle/product/19c
mkdir -p /u01/app/grid/19c
mkdir -p /u01/oraInventory
mkdir -p /oraarch

chown -R oracle:dba /u01
chmod -R 775 /u01

chown -R oracle:dba /oraarch
chmod -R 775 /oraarch

chown -R oracle:dba /dev
```

---

### 3. ASMLib 설치

- 공유 디스크를 ASM으로 구성할 것이므로 ASMLib를 설치한다.

<aside>
📌

사이트 접속 후 `Ctrl+F`로 파일명을 검색하면 빠르게 찾을 수 있다.

</aside>

- `oracleasmlib-2.0.17-1.el8.x86_64.rpm`
🔗 [https://www.oracle.com/linux/downloads/linux-asmlib-v8-downloads.html](https://www.oracle.com/linux/downloads/linux-asmlib-v8-downloads.html)

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image.png)

- `oracleasm-support-2.1.12-1.el8.x86_64.rpm`
🔗 [https://yum.oracle.com/repo/OracleLinux/OL8/addons/x86_64/index.html](https://yum.oracle.com/repo/OracleLinux/OL8/addons/x86_64/index.html)

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/49d9e2a5-8240-49d1-9630-712db791c222.png)

#### RPM 다운로드 후 업로드

- 위 두 사이트에서 rpm 파일을 각각 다운로드한 뒤, MobaXterm으로 리눅스에 업로드한다.

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image%201.png)

#### 설치

```bash
[root@rac1 ~]# rpm -ivh oracleasmlib-2.0.17-1.el8.x86_64.rpm
[root@rac1 ~]# rpm -ivh oracleasm-support-2.1.12-1.el8.x86_64.rpm
```

---

### 4. 공유 메모리(shmmax, shmall) 값 설정

- `/etc/sysctl.conf`의 `shmmax`, `shmall` 값을 메모리의 절반(4GB = 4294967296 바이트)로 설정한 뒤 반영한다.

```bash
[root@rac1 ~]# vi /etc/sysctl.conf
kernel.shmall=4294967296
kernel.shmmax=4294967296
```

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image%202.png)

```bash
[root@rac1 ~]# /sbin/sysctl -p

shmmax = 단일 프로세스가 공유 메모리를 호출하기 위한 최대 값
shmall = 모든 프로세스가 사용할 수 있는 총 공유 메모리 값
```

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image%203.png)

---

### 5. root 사용자 `.bash_profile` 수정

- root 사용자의 `.bash_profile`에 Oracle 및 Grid 관련 환경 변수를 추가한다.

```bash
[root@rac1 ~]# vi .bash_profile

export ORACLE_BASE=/u01/app/oracle;
export GRID_HOME=/u01/app/grid/19c;
export ORACLE_HOME=$ORACLE_BASE/product/19c;
export PATH=$PATH:$GRID_HOME/bin;

[root@rac1 ~]# source .bash_profile
```

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image%204.png)

---

### 6. oracle 사용자 `.bash_profile` 수정

- root 사용자에서 oracle 사용자의 `.bash_profile`을 수정한다. (또는 oracle 사용자로 접속해 수정)

```bash
[root@rac1 ~]# vi ~oracle/.bash_profile

# 아래 내용 추가
export ORACLE_BASE=/u01/app/oracle;
export ORACLE_HOME=$ORACLE_BASE/product/19c;
export ORACLE_SID=rac1;        # 2번 노드: rac2
export ORACLE_HOSTNAME=rac1;   # 2번 노드: rac2
export ORACLE_UNQNAME=racdb;
export GRID_HOME=/u01/app/grid/19c;
export GRID_SID=+ASM1;         # 2번 노드: ASM2
export PATH=$ORACLE_HOME/bin:$GRID_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib;
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib;
export EDITOR=vi;

alias grid='export ORACLE_HOME=$GRID_HOME; export ORACLE_SID=$GRID_SID; export PATH=$ORACLE_HOME/bin:$GRID_HOME/bin:$PATH; echo $ORACLE_SID; echo $ORACLE_HOME'
alias db='. ~oracle/.bash_profile;export PATH=$ORACLE_HOME/bin:$GRID_HOME/bin:$PATH; echo $ORACLE_SID;echo $ORACLE_HOME'
alias oh='cd $ORACLE_HOME;pwd'
alias ss='sqlplus / as sysdba'

[root@rac1 ~]# source ~oracle/.bash_profile
```

![image.png](%EA%B7%B8%EB%A6%AC%EB%93%9C%20%EC%84%A4%EC%B9%98%EC%A0%84%20%EB%A6%AC%EB%88%85%EC%8A%A4%20%ED%99%98%EA%B2%BD%EC%84%A4%EC%A0%95/image%205.png)

#### oracle 사용자 전환 확인

- 아래와 같이 oracle 사용자로 접속 가능하면 설정은 정상이다.

```bash
[root@rac1 ~]# su - oracle
[oracle@rac1 ~]$
```