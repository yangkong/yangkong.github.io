---
title: "Bacula Manual"
categories:
  - Backup
  - Linux
tags:
  - backup
  - bacula
---

# 백업 시스템 (Bacula) 매뉴얼
> 오픈소스 백업 시스템 Bacula (v9.6.3) 설치 및 운영 매뉴얼

## Bacula 구조
![enter image description here](https://i.imgur.com/3reJykC.png)

- 디렉터 데몬 [시스템 전체를 컨트롤]
- 스토리지 데몬 [백업 파일 저장소]
- 파일 데몬 [백업 대상]
- 데이터베이스 [백업 관련 데이터 저장(카탈로그)]

## Config 구조
![enter image description here](https://i.imgur.com/IwxPfBH.png)

## 설치
> Cent OS 7 / MariaDB 기준 

1. Import the GPG key

    ```
    cd /tmp
    ```
    ```
    wget https://www.bacula.org/downloads/Bacula-4096-Distribution-Verification-key.asc
    ```
    ```
    rpm --import Bacula-4096-Distribution-Verification-key.asc
    ```
    ```
    rm Bacula-4096-Distribution-Verification-key.asc
    ```

2. Bacula.repo

	```
	vi /etc/yum.repos.d/Bacula.repo
	```

	아래내용 작성
	```
	[Bacula-Community]  
	name=CentOS - Bacula - Community  
	baseurl=http://www.bacula.org/packages/@access-key@/rpms/@bacula-version@/el7/x86_64/  
	enabled=1  
	protect=0  
	gpgcheck=1
	```
	@access-key@ = 5ec2xxxx3665 (bacula 홈페이지 이메일 제출후 얻을수있음)  
	@bacula-version@ = 버전 (ex : 9.6.3)

3. 설치
	```
	yum install bacula-mysql.x86_64
	```
	* 스토리지 데몬만 설치할 경우
	```
	yum install bacula-storage.x86_64
	```
	* 파일 데몬만 설치할 경우
	```
	yum install bacula-client.x86_64
	```

4. 카탈로그 데이터베이스 설정
	```
	sh /opt/bacula/scripts/create_mysql_database
	```
	```
	sh /opt/bacula/scripts/make_mysql_tables
	```
	```
	sh /opt/bacula/scripts/grant_mysql_privileges
	```
	> grant_mysql_privileges 스크립트의 경우 작동이 안될 경우가 있는데
	해당스크립트 안에 db_password 변수에 패스워드를 직접 입력해야한다.

	스크립트 실행 후 데이터베이스에 접속하여 bacula 스키마, 테이블, 유저가 제대로 생성 되었는지 확인한다.

5. 포트 오픈 확인
	> 각 데몬이 서버가 분리되어 운영 된다면 해당 서버에서 아래의 포트를 OPEN 
	
	9101 - 디렉터 데몬
	9102 - 스토리지 데몬
	9103 - 파일 데몬
	
## Config 파일
![enter image description here](https://i.imgur.com/hs2nR09.png)

### 디렉터 Config
> File 위치 
```
/opt/bacula/etc/bacula-dir.conf
```
> 원문 링크 [https://ring_Director.html](https://uals/en/main/Configuring_Director.html)

- - -
**Director**
> 디렉터 데몬 설정

```
Director {
  Name = bacula-dir
  DIRport = 9101
  QueryFile = "/opt/bacula/scripts/query.sql"
  WorkingDirectory = "/opt/bacula/working"
  PidDirectory = "/opt/bacula/working"
  Maximum Concurrent Jobs = 20
  Password = "Passw0rd"
  Messages = Daemon
}
```
* Name : 디렉터 이름 (다른곳 참조 할때 사용)
* DIRport : 포트 번호 (기본 9101)
* Password : Bacula 콘솔 Config에서 필요한 패스워드 (설치시 임의 문자를 제공함)
* Messages : Messages 설정값 Name
- - -

**Job**
> 백업 및 복원등의 작업 설정

```
Job {
  Name = "BackupJob"
  Type = Backup
  Level = Incremental
  Client = localhost-fd
  FileSet = "Full Set"
  Schedule = "WeeklyCycle"
  Storage = File1
  Messages = Standard
  Pool = File
  Priority = 10
  Write Bootstrap = "/opt/bacula/working/%c.bsr"
}
```
* Name : (다른곳 참조 할때 사용)
* Type : 작업 종류 (Backup/Restore/Verify 등등)
* Level : 작업 레벨 (Full/Incremental/Differential 등등)
* Client : 파일데몬 Name
* FileSet : FileSet 설정값 Name
* Schedule : Schedule 설정값 Name
* Storage : Storage 또는 Autochanger 설정값 Name
* Messages : Messages 설정값 Name
* Pool : Pool 설정값 Name
* Priority : 우선순위
* Write Bootstrap : 부트스트랩 파일 지정

- - -
**JobDefs**
> Job과 동일한 형식으로 설정하며 Job에서 JobDefs로 연결하여 사용 (템플릿 개념)

- - -
**FileSet**
> Job 설정에서 사용하며 백업 파일 상세 설정 

```
FileSet {
  Name = "Home Set"
  Include {
    Options {
      signature = MD5
      compression = GZIP
    }
    File = /home
  }
  Exclude {
    File = /opt/bacula/working
    File = /tmp
  }
}
```
* Include : 백업에 포함될 파일 설정
* Exclude : 백업에 포함되지 않을 파일 설정 
* File : 디렉토리 또는 파일
* signature : 백업 파일에 서명 알고리즘 (Option)
* compression : 압축 방식 (Option)

- - -
**Schedule**
> 백업 스케쥴 설정

```
Schedule {
  Name = "WeeklyCycle"
  Run = Full 1st sun at 23:05
  Run = Differential 2nd-5th sun at 23:05
  Run = Incremental mon-sat at 23:05
}
```
* Run : 동작 레벨 과 일정 키워드 조합

```
동작 키워드 표
<void-keyword>    = on
<at-keyword>      = at
<week-keyword>    = 1st | 2nd | 3rd | 4th | 5th | 6th | first |
                    second | third | fourth | fifth | sixth
<wday-keyword>    = sun | mon | tue | wed | thu | fri | sat |
                    sunday | monday | tuesday | wednesday |
                    thursday | friday | saturday
<week-of-year-keyword> = w00 | w01 | ... w52 | w53
<month-keyword>   = jan | feb | mar | apr | may | jun | jul |
                    aug | sep | oct | nov | dec | january |
                    february | ... | december
<daily-keyword>   = daily
<weekly-keyword>  = weekly
<monthly-keyword> = monthly
<hourly-keyword>  = hourly
<digit>           = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 0
<number>          = <digit> | <digit><number>
<12hour>          = 0 | 1 | 2 | ... 12
<hour>            = 0 | 1 | 2 | ... 23
<minute>          = 0 | 1 | 2 | ... 59
<day>             = 1 | 2 | ... 31 | lastday
<time>            = <hour>:<minute> |
                    <12hour>:<minute>am |
                    <12hour>:<minute>pm
<time-spec>       = <at-keyword> <time> |
                    <hourly-keyword>
<date-keyword>    = <void-keyword>  <weekly-keyword>
<day-range>       = <day>-<day>
<month-range>     = <month-keyword>-<month-keyword>
<wday-range>      = <wday-keyword>-<wday-keyword>
<range>           = <day-range> | <month-range> |
                          <wday-range>
<date>            = <date-keyword> | <day> | <range>
<date-spec>       = <date> | <date-spec>
<day-spec>        = <day> | <wday-keyword> |
                    <day> | <wday-range> |
                    <week-keyword> <wday-keyword> |
                    <week-keyword> <wday-range> |
                    <daily-keyword>
<month-spec>      = <month-keyword> | <month-range> |
                    <monthly-keyword>
<date-time-spec>  = <month-spec> <day-spec> <time-spec>
```

- - -
**Client**
> 파일 데몬 설정

```
Client {
  Name = localhost-fd
  Address = localhost
  FDPort = 9102
  Catalog = MyCatalog
  Password = "0eZafLb1Zqs1J3WJ46sV12L7fJ/G6B20GP5RFei1a09v"
  File Retention = 60 days
  Job Retention = 6 months
  AutoPrune = yes
}
```
* Address : 파일 데몬 서버 IP
* FDPort : 파일 데몬 서버 포트
* Catalog : 카탈로그 설정 Name
* Password : 파일 데몬 서버의 Config 에서 사용할 Password
* File Retention : 카탈로그 데이터베이스에서 파일 레코드의 보존기간 설정
* Job Retention : 카탈로그 데이터베이스에서 작업 레코드의 보존기간 설정
* AutoPrune : 카탈로그 데이터베이스에서 각 보존기간에 따른 자동 정리 설정
 
- - -
**Autochanger**
> 스토리지 데몬 설정
> 스토리지 데몬 Config 값과 연결 되어있음

```
Autochanger {
  Name = File1
  Address = 192.168.99.6
  SDPort = 9103
  Password = "TBTe/sdH0gaZ3sN12DS1NGPUJWA/NkRgEJPJtNdCv3a+UM"
  Device = FileChgr1
  Media Type = File1
  Maximum Concurrent Jobs = 10
  Autochanger = File1
}
```
* Address : 스토리지 데몬 IP
* SDPort : 스토리지 데몬 포트
* Password : 스토리지 데몬 서버의 Config 에서 사용할 Password
* Device : 스토리지 데몬 서버의 Config 에서 Autochanger Name
* Media Type : 스토리지 데몬 서버의 Config 에서 Autochanger의 Device 설정의 Media Type과 동일하게 입력
* Autochanger : point to ourself

- - -
**Pool**
> Bacula가 데이터를 쓰는 데 사용할 스토리지 볼륨 세트 (테이프 또는 파일)를 정의

```
Pool {
  Name = File
  Pool Type = Backup
  Recycle = yes
  AutoPrune = yes
  Volume Retention = 365 days
  Maximum Volume Bytes = 50G
  Maximum Volumes = 100
  Label Format = "Vol-"
}
```
* Pool Type : 현재 Backup Type만 지원 
* Recycle : 볼륨 재활용 여부
* AutoPrune : 카탈로그 자동 정리 여부
* Volume Retention : 볼륨 보존 기간
* Maximum Volume Bytes : 최대 볼륨 사이즈
* Maximum Volumes : 최대 볼륨 갯수
* Label Format : 볼륨 라벨 지정

- - -
**Catalog**
> 카탈로그 데이터베이스 설정 (Maria DB)

```
Catalog {
  Name = MyCatalog
  dbname = "bacula"; dbuser = "bacula"; dbpassword = "password"
}
```

### 파일 데몬(클라이언트) Config
> File 위치 ```/opt/bacula/etc/bacula-fd.conf```
> 원문 링크 [https://www.bacula.org/9.6.x-manuals/en/main/Client_File_daemon_Configur.html](https://www.bacula.org/9.6.x-manuals/en/main/Client_File_daemon_Configur.html)
- - -

**Director**
> 디렉터 데몬 설정

```
Director {
  Name = bacula-dir
  Password = "0eZafLb1Zqs1JYWJ46sVnoL7fJ/G6B206c95RFei1a09v"
}
```
* Name : 디렉터 Config의 Director Name
* Password : 디렉터 Config의 Director Password

- - -

**FileDaemon**
> 파일 데몬 설정

```
FileDaemon {
  Name = localhost-fd
  FDport = 9102
  WorkingDirectory = /opt/bacula/working
  Pid Directory = /opt/bacula/working
  Maximum Concurrent Jobs = 20
  Plugin Directory = /opt/bacula/plugins
}
```
* FDport : 데몬 포트


### 스토리지 데몬(저장소) Config
> File 위치 ```/opt/bacula/etc/bacula-sd.conf```
> 원문 링크 [https://www.bacula.org/9.6.x-manuals/en/main/Storage_Daemon_Configuratio.html](https://www.bacula.org/9.6.x-manuals/en/main/Storage_Daemon_Configuratio.html)

- - -

**Director**
> 디렉터 데몬 설정

```
Director {
  Name = bacula-dir
  Password = "0eZafLb1Zqs1JYWJ46sVnoL7fJ/G6B206c95RFei1a09v"
}
```
* Name : 디렉터 Config의 Director Name
* Password : 디렉터 Config의 Director Password

- - -

**Storage**
> 스토리지 데몬 설정

```
Storage {
  Name = localhost-sd
  SDPort = 9103
  WorkingDirectory = "/opt/bacula/working"
  Pid Directory = "/opt/bacula/working"
  Plugin Directory = "/opt/bacula/plugins"
  Maximum Concurrent Jobs = 20
}
```
* SDPort : 데몬 포트

- - -

**Autochanger**
> 디렉터 Config와 연동되는 설정

```
Autochanger {
  Name = FileChgr1
  Device = FileChgr1-6_home, FileChgr1-Dev2
  Changer Command = ""
  Changer Device = /dev/null
}
```
* Device : Device 설정 Name

- - -

**Device**
> 저장 장치 설정

```
Device {
  Name = FileChgr1-6_home
  Media Type = File1
  Archive Device = /data1/bacula/6_home
  Random Access = Yes;
  AutomaticMount = yes;
  RemovableMedia = no;
  Maximum Concurrent Jobs = 5
}
```
* Media Type : 저장 장치 임의 설명
* Archive Device : 저장 디렉토리
* Random Access : 저장 장치의 랜덤 액세스 지원여부 


## 시스템 실행 
### 실행 전 Config 파일 테스트
```
cd /opt/bacula/bin
./bacula-dir -t -c ../etc/bacula-dir.conf
./bacula-fd -t -c ../etc/bacula-fd.conf
./bacula-sd -t -c ../etc/bacula-sd.conf
./bconsole -t -c ../etc/bconsole.conf
```
- - -
### 데몬 실행 | 재시작 | 정지 | 상태
```
cd /opt/bacula/script
./bacula-ctl-dir start|restart|stop|status
./bacula-ctl-fd start|restart|stop|status
./bacula-ctl-sd start|restart|stop|status
```

## 시스템 운영 
> bconsole 을 이용하여 전체적인 컨트롤
> 콘솔 실행 
```
/opt/bacula/script/bconsole
```
> 콘솔 접속 후 help 명령어로 명령어 일람/ Tab으로 자동 완성 기능 제공

### 백업
> Config 값에 의해 스케쥴에 따라 자동으로 백업 진행

**수동 백업**
1. `run` 명령어 입력
2. Config에 설정된 Job 리스트 출력 / 해당 번호 입력
3. Config에 설정된 Job 설정값 출력 / yes/mod/no (실행/수정/취소) 선택
4. 실행시 백업 Job이 Queue에 추가됨
5. `status dir` 명령어로 상태 확인
6. 백업 완료 혹은 오류 시 message가 발생하며 `messages` 명령어로 확인


- - -
### 복원
> Config 값에 복원 Job을 설정 후 진행

1. `restore` 명령어 입력
2. 복원 관련 선택지 중 5번 선택 (파일 데몬 선택 및 복원 최근 지점 자동 선택)
3. 복원 대상 파일 선택 (`ls`로 확인 후 `mark` 명령어로 선택 후 `done `명령어로 완료)
4. Config에 설정된 Job 중 Restore 타입인 리스트 출력 / 해당 번호 입력
5. Config에 설정된 Job 설정값 출력 / yes/mod/no (실행/수정/취소) 선택
6. 실행시 복원 Job이 Queue에 추가됨
7. `status dir` 명령어로 상태 확인
8. 백업 완료 혹은 오류 시 message가 발생하며 `messages` 명령어로 확인


## 참조
* [https://www.bacula.org/](https://www.bacula.org/)
* [https://www.bacula.org/documentation/documentation/](https://www.bacula.org/documentation/documentation/)
* [https://www.bacula.org/bacula-binary-package-download/](https://www.bacula.org/bacula-binary-package-download/)
