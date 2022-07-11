# 프로젝트명
> 본 프로젝트는 플래시 메모리 상에서의 파일 시스템에 따른 DBMS 성능을 실험한다. 또한, discard 명령이 파일 시스템의 성능에 끼치는 영향을 연구한다.

플래시 메모리에서 FFS(Ext4, XFS)와 LFS(F2FS)의 사용에 따른 DBMS(MySQL/RocksDB) OLTP(벤치마크 툴: TPC-C, YCSB) 성능을 비교한다.

![](../header.png)

## 배경 지식

### 플래시 메모리

윈도우:

### MySQL
MySQL은 가장 많이 쓰이는 오픈 소스 RDBMS이다. MySQL서버는 크게 MySQL 엔진과 스토리지 엔진으로 나누어지는데, MySQL 엔진은 SQL 문장을 분석, 최적화등 클라이언트로부터 오는 요청을 처리하고, 스토리지 엔진은 실제 데이터를 디스크 스토리지에 저장하거나 조회한다. MySQL에 다양한 스토리지 엔진을 사용 가능하지만 그 중에서 B+ Tree기반의 InnoDB 스토리지 엔진이 가장 널리 쓰인다. InnoDB는 메모리 영역과 디스크 영역으로 나누어 진다. 사용자가 DB를 변경한다면 in-place-update방식으로 디스크 영역에 데이터가 쓰인다.

Reference: https://jeong-pro.tistory.com/239

### TPC-C
TPC란 Transaction Processing Performance Council 에서 발표한 벤치마크 모델들이다. TCP는 OLTP 시스템의 처리 성능을 평가하는 기준이 된다. 그 중 TPC-C는 TPC-A 모델이나 TPC-B 모델보다 복잡한 유통업의 수주·발주 OLTP 성능 평가를 위한 벤치마크 모델이다.

### RocksDB
RocksDB는 SSD에 최적화된 Key-Value 형태의 로그 구조 데 이터베이스 엔진이다. RocksDB의 데이터 저장 구조는 Log-Structured Merge Tree(LSM-tree)를 기반으로 한다. RocksDB는 쓰기 요청 시 메모리의 Active Memtable이란 이름의 임시 버퍼에 데이터를 쓴다. 해당 Memtable이 일정 크기가 되면 읽기 전용 Memtable이 되고, 일정 개수 이상의 Memtable이 모이면 저장 장치에 내려가(flush) SST 파일 형태로 저장된다. 따라서 RocksDB는 SST 파일 단위로(기본 64MB) I/O를 진행하고, 데이터를 append-only 로그에 저장함으로써 순차 쓰기를 보장한다.

### YCSB
YCSB(Yahoo! Cloud Serving Benchmark)는 클라우드 서비스와 NoSQL의 성능을 측정하기위한 벤치마크 툴이다. YCSB는 기본적으로 제공하는 다른 작동방식을 가진 workload들이 있다. 또 사용자가 이를 수정하거나 직접 만들어 사용할 수도 있다.
### 파일 시스템

### 성능 측정 지표


## 실험 환경

스크린 샷과 코드 예제를 통해 사용 방법을 자세히 설명합니다.

_더 많은 예제와 사용법은 [Wiki][wiki]를 참고하세요._

## 쉘 사용법

모든 개발 의존성 설치 방법과 자동 테스트 슈트 실행 방법을 운영체제 별로 작성합니다.

```sh
make install
npm test
```

## 결과 분석 방법


## 프로젝트 정보

학번 - 이름 - github주소 - 이메일

## SSD 초기화 및 File System Mount
0. Data directory 내용 삭제
```sh
rm -rf /path/to/datadir/*
```

1. 기존 SSD Unmount
```sh
sudo umount /dev/[PARTITION]
```

2. SSD Blksidcard
```sh
sudo blkdiscard /dev/[DEVICE]
```

3. 파디션 생성
```sh
sudo fdisk /dev/[DEVICE]
//command: n (new partition), w (write, save)
//Whole SSD in 1 partition
```

4. File system 생성
```sh
//F2FS
sudo mkfs.f2fs /dev/[PARTITION] -f

//Ext4
sudo mkfs.ext4 /dev/[PARTITION] -E discard,lazy_itable_init=0,lazy_journal_init=0 -F

//XFS
sudo mkfs.xfs /dev/[PARTITION] -f
```

5. Data directory에 mount
```sh
//F2FS default
sudo mount /dev/[PARTITION] /path/to/datadir 

//F2FS lfs mode
sudo mount -o mode=lfs /dev/[PARTITION] /path/to/datadir 

///Ext4, XFS
sudo mount -o discard /dev/[PARTITION] /path/to/datadir 
```

6. 권한 설정
```sh
sudo chown -R USER[:GROUP] /path/to/datadir 
sudo chmod -R 777 /path/to/datadir
```

## MySQL , TPC-C 실험
1. Install MySQL 5.7 and TPC-C  
Reference the [installation guide](https://github.com/meeeejin/SWE3033-F2021/blob/main/week-1/reference/tpcc-mysql-install-guide.md) to install and run TPC-C benchmark on MySQL 5.7

2. my.cnf 수정
```sh
#
# The MySQL database server configuration file
#
[client]
user    = root
port    = 3306
socket  = /tmp/mysql.sock

[mysql]
prompt  = \u:\d>\_

[mysqld_safe]
socket  = /tmp/mysql.sock

[mysqld]
# Basic settings
default-storage-engine = innodb
pid-file        = /path/to/datadir/mysql.pid
socket          = /tmp/mysql.sock
port            = 3306
datadir         = /path/to/datadir/
log-error       = /path/to/datadir/mysql_error.log

#
# Innodb settings
#
# Page size
innodb_page_size=16KB

# Buffer pool settings

##### Change buffer pool size according to data size(5GB, 10GB, 15GB)
innodb_buffer_pool_size=5G
innodb_buffer_pool_instances=8

# Transaction log settings
innodb_log_file_size=100M
innodb_log_files_in_group=2
innodb_log_buffer_size=32M

# Log group path (iblog0, iblog1)
# If you separate the log device, uncomment and correct the path
#innodb_log_group_home_dir=/path/to/logdir/

# Flush settings (SSD-optimized)
# 0: every 1 seconds, 1: fsync on commits, 2: writes on commits
innodb_flush_log_at_trx_commit=0
innodb_flush_neighbors=0
# innodb_flush_method=O_DIRECT

##### F2FS lfs mode O_DIRECT error
innodb_flush_method=fsync
```

3. 재실험 시 data directory 및 MySQL 초기화

```sh
//서버 종료
./bin/mysqladmin -uroot -p[yourPassword] shutdown

//SSD 초기화 및 File System Mount
...
...
//


./bin/mysqld --initialize --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir

./bin/mysqld_safe --skip-grant-tables --datadir=/path/to/datadir

./bin/mysql -uroot

root:(none)> use mysql;

root:mysql> update user set authentication_string=password('yourPassword') where user='root';
root:mysql> flush privileges;
root:mysql> quit;

./bin/mysql -uroot -p[yourPassword]

root:mysql> set password = password('yourPassword');
root:mysql> quit;

./bin/mysqladmin -uroot -pyourPassword shutdown
./bin/mysqld_safe --defaults-file=/path/to/my.cnf

./bin/mysql -u root -p[yourPassword] -e "CREATE DATABASE tpcc;"
./bin/mysql -u root -p[yourPassword] tpcc < /home/vldb/MySQL/tpcc-mysql/create_table.sql
./bin/mysql -u root -p[yourPassword] tpcc < /home/vldb/MySQL/tpcc-mysql/add_fkey_idx.sql
```
4. TPC-C, 로그 쉘 시작
```sh
./log_write.sh

./tpcc_start -h 127.0.0.1 -S /tmp/mysql.sock -d tpcc -u root -p "yourPassword" -w [Warehouse Number] -c 8 -r 10 -l [Run Time] | tee [Experiment Name].txt

```
## RocksDB, YCSB 실험

<!-- Markdown link & img dfn's -->
[npm-image]: https://img.shields.io/npm/v/datadog-metrics.svg?style=flat-square
[npm-url]: https://npmjs.org/package/datadog-metrics
[npm-downloads]: https://img.shields.io/npm/dm/datadog-metrics.svg?style=flat-square
[travis-image]: https://img.shields.io/travis/dbader/node-datadog-metrics/master.svg?style=flat-square
[travis-url]: https://travis-ci.org/dbader/node-datadog-metrics
[wiki]: https://github.com/yourname/yourproject/wiki
