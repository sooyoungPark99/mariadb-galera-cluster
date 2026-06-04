# MariaDB 10.6 Galera Cluster 구성

3노드 Galera Cluster를 단계별로 구성한다.

---

## 단계별 수행 노드 요약

| 단계 | 작업 | 수행 노드 |
|------|------|----------|
| 1-1 | VM 복제 | VirtualBox (호스트) |
| 1-2 | 호스트명/IP 변경 | galera1, galera2, galera3 각각 |
| 1-3 | /etc/hosts 등록 | 전 노드 동일 |
| 1-4 | 데이터 초기화 | 전 노드 |
| 2-1 | galera.cnf 설정 | 전 노드 (노드별 값 일부 변경) |
| 3-1 | 부트스트랩 | galera1만 |
| 3-2 | 순차 기동 | galera2, galera3 |
| 3-3 | 클러스터 크기 확인 | 아무 노드 |
| 4 | 클러스터 테스트 | 전 노드 |

> 시스템 작업(vi, systemctl, rm 등)은 OS root에서, DB 작업(mysql 접속 후 SQL)은 MariaDB root 계정으로 수행한다.

---

## 1. VM 복제 및 사전 준비

### 1-1. VM 복제 (VirtualBox)

MariaDB Single 설치가 완료된 VM을 VirtualBox에서 복제하여 3대를 구성한다.

1. VirtualBox에서 기존 MariaDB VM 우클릭 → 복제
2. 복제 방식: 완전한 복제 (Full Clone)
3. MAC 주소 정책: 모든 네트워크 어댑터의 새 MAC 주소 생성
4. 위 과정을 반복하여 총 3대 구성 (galera1, galera2, galera3)

### 1-2. 호스트명 및 IP 변경 (각 노드에서)

> 복제본은 호스트명/IP가 동일하므로 노드마다 다르게 설정한다.

**galera1 노드에서:**

```bash
hostnamectl set-hostname galera1
nmcli con mod eth0 ipv4.addresses 172.31.0.253/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```
<img width="972" height="682" alt="image" src="https://github.com/user-attachments/assets/a9baed50-be46-4826-bdd9-0825c65c7c41" />

**galera2 노드에서:**

```bash
hostnamectl set-hostname galera2
nmcli con mod eth0 ipv4.addresses 172.31.0.254/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```
<img width="976" height="685" alt="image" src="https://github.com/user-attachments/assets/d4570b49-43c6-4127-b0f2-12b611ac7930" />

**galera3 노드에서:**

```bash
hostnamectl set-hostname galera3
nmcli con mod eth0 ipv4.addresses 172.31.0.255/16
nmcli con mod eth0 ipv4.gateway 172.31.0.1
nmcli con down eth0 && nmcli con up eth0
```
<img width="972" height="683" alt="image" src="https://github.com/user-attachments/assets/625b8e8e-a9d0-4d02-9e42-ac71ddcfbea4" />

#### -> GUI(설정 → 네트워크)로 설정해도 결과는 동일하다.
---
### 1-3. /etc/hosts 등록 (전 노드 동일)

> 호스트명으로 서로 통신할 수 있도록 전 노드에 동일하게 등록한다.

```bash
vi /etc/hosts
```

```
172.31.0.253  galera1
172.31.0.254  galera2
172.31.0.255  galera3
```
<img width="1108" height="168" alt="image" src="https://github.com/user-attachments/assets/5d095683-fc27-494d-bd4d-f76ca03102d5" />

### 1-4. 복제본 데이터 초기화 (전 노드)

> 클러스터는 빈 상태에서 첫 노드 데이터를 전체 노드에 동기화한다. 복제본에 남아있는 기존 데이터를 비우고 시작한다.

```bash
systemctl stop mariadb
rm -rf /var/lib/mysql/*
mysql_install_db --user=mysql --datadir=/var/lib/mysql
```

> 클러스터 기동 전이므로 아직 mariadb를 start하지 않는다.

---

## 2. Galera 설정 (전 노드)

> 이 섹션은 galera1, galera2, galera3 전 노드에서 수행한다.
> 단, `wsrep_node_name`과 `wsrep_node_address`는 노드마다 다르게 설정한다.

### 2-1. Galera 설정 파일 작성 (전 노드)

> MariaDB 10.6은 Galera(galera-4)가 기본 포함되어 별도 설치가 필요 없다. 설정만 추가하면 된다.

```bash
vi /etc/my.cnf.d/galera.cnf
```

```
[galera]
wsrep_on = ON
wsrep_provider = /usr/lib64/galera-4/libgalera_smm.so
wsrep_cluster_name = "maria_galera_cluster"
wsrep_cluster_address = "gcomm://172.31.0.253,172.31.0.254,172.31.0.255"

binlog_format = ROW
default_storage_engine = InnoDB
innodb_autoinc_lock_mode = 2

# 노드별로 아래 2개 값을 각자 변경한다
wsrep_node_name = "galera1"
wsrep_node_address = "172.31.0.253"
```
<img width="1060" height="306" alt="image" src="https://github.com/user-attachments/assets/b5ffa737-3dde-4a1f-88a8-d60139a25e37" />
<img width="1857" height="310" alt="image" src="https://github.com/user-attachments/assets/3a369a56-ef72-47df-a2be-9f62ce4b781d" />

| 항목 | 설명 | 노드별 차이 |
|------|------|-----------|
| wsrep_on | Galera 복제 활성화 | 동일 |
| wsrep_provider | Galera 라이브러리 경로 | 동일 |
| wsrep_cluster_name | 클러스터 이름 | 전 노드 동일해야 함 |
| wsrep_cluster_address | 클러스터 구성 노드 IP 목록 | 동일 |
| binlog_format | ROW 필수. Galera는 ROW만 지원 | 동일 |
| innodb_autoinc_lock_mode | 2 필수. AUTO_INCREMENT 충돌 방지 | 동일 |
| wsrep_node_name | 노드 고유 이름 | **노드마다 다름** |
| wsrep_node_address | 해당 노드의 IP | **노드마다 다름** |

**노드별 변경값:**

| 노드 | wsrep_node_name | wsrep_node_address |
|------|-----------------|--------------------|
| galera1 | "galera1" | "172.31.0.253" |
| galera2 | "galera2" | "172.31.0.254" |
| galera3 | "galera3" | "172.31.0.255" |

> 마지막 2개 항목만 노드별로 바꾸고, 나머지는 전 노드 동일하게 설정한다.

---

## 3. 클러스터 기동

### 3-1. 첫 번째 노드 부트스트랩 (galera1에서만)

> 클러스터를 처음 생성할 때는 기준이 되는 첫 노드를 부트스트랩 모드로 기동한다. 이 노드가 클러스터의 최초 멤버가 된다.

```bash
# galera1에서만 실행
galera_new_cluster
```

상태 확인:

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

```
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
```
<img width="1142" height="165" alt="image" src="https://github.com/user-attachments/assets/30979e2d-1bc1-4641-97c6-681c765c126b" />

> 부트스트랩은 첫 노드를 처음 기동할 때 한 번만 사용한다. 이후에는 일반 `systemctl start mariadb`로 기동한다.

### 3-2. 나머지 노드 순차 기동 (galera2 → galera3 순서)

> 부트스트랩된 첫 노드에 나머지 노드가 순차적으로 합류한다. 합류 시 첫 노드의 데이터가 자동으로 동기화된다.

```bash
# galera2에서 실행
systemctl start mariadb
```

```bash
# galera3에서 실행
systemctl start mariadb
```

### 3-3. 클러스터 크기 확인 (아무 노드에서)

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

```
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
```

> `wsrep_cluster_size = 3`이면 3노드 클러스터 구성 완료다.

클러스터 상태도 확인한다.

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_status';"
```

```
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_status | Primary |
+----------------------+---------+
```

<img width="915" height="341" alt="image" src="https://github.com/user-attachments/assets/4f089081-2ead-4e4c-b3af-6e63f4a18ea6" />

---

## 4. 클러스터 테스트

### 4-1. galera1에서 데이터 생성 (galera1)

```bash
mysql -u root -p
```

```sql
CREATE DATABASE galeratest;
USE galeratest;

CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(20));
INSERT INTO t1 VALUES (1, 'node1-data');

EXIT;
```

### 4-2. galera2에서 동기화 확인 및 쓰기 (galera2)

> Multi-Primary이므로 galera2에서도 쓰기가 가능하다.

```bash
mysql -u root -p
```

```sql
USE galeratest;
SELECT * FROM t1;
-- (1, 'node1-data') 조회되면 동기화 성공

-- galera2에서도 쓰기 가능
INSERT INTO t1 VALUES (2, 'node2-data');
EXIT;
```

### 4-3. galera3에서 전체 데이터 확인 (galera3)

```bash
mysql -u root -p
```

```sql
USE galeratest;
SELECT * FROM t1;
-- (1, 'node1-data'), (2, 'node2-data') 모두 조회되면 정상
EXIT;
```
<img width="1862" height="405" alt="image" src="https://github.com/user-attachments/assets/02d88ed1-9ab9-4d11-a46f-e9dc783af3ae" />

> 모든 노드에서 동일한 데이터가 조회되면 Multi-Primary 동기 복제가 정상 동작하는 것이다.

---

## 5. 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| 노드 합류 실패 | 방화벽 차단 | 4567, 4568, 4444 포트 확인 (방화벽 사용 시) |
| wsrep_cluster_size = 1 유지 | 노드 간 통신 불가 | /etc/hosts, IP, 네트워크 확인 |
| 부트스트랩 노드 기동 실패 | grastate.dat의 safe_to_bootstrap=0 | 해당 파일에서 1로 변경 후 재시도 |
| 전체 노드 다운 후 재기동 | 마지막 종료 노드부터 부트스트랩 필요 | seqno가 가장 큰 노드에서 galera_new_cluster |
| Split-Brain | 노드 과반수 미달 | 최소 과반수 노드 정상화 |

### Galera 사용 포트

| 포트 | 용도 |
|------|------|
| 3306 | MariaDB 일반 접속 |
| 4567 | Galera 노드 간 통신 (복제 트래픽) |
| 4568 | IST (증분 상태 전송) |
| 4444 | SST (전체 상태 전송) |

> 실습 환경에서 방화벽이 비활성화되어 있으면 포트 설정은 생략한다.

### 전체 노드 종료 후 재기동 순서

> 클러스터 전체가 종료되면 마지막으로 종료된 노드가 가장 최신 데이터를 가진다. 이 노드부터 부트스트랩해야 데이터 손실이 없다.

```bash
# 각 노드에서 마지막 상태 확인
cat /var/lib/mysql/grastate.dat

# seqno 값이 가장 큰 노드에서 부트스트랩
galera_new_cluster

# 나머지 노드는 일반 기동
systemctl start mariadb
```
---
 
### MobaXterm 연결 실패 (Connection timed out / Cannot assign requested address)
 
**증상:** VM IP/SSH 모두 정상인데 MobaXterm에서 연결 실패
 
**원인:** Windows PC에 네트워크 어댑터가 여러 개(물리 NIC + VMware/VPN 가상 어댑터 등) 있을 때, VM의 ARP가 잘못된 어댑터에 잡혀 라우팅 불일치가 발생한다.
 
**진단 순서:**
 
1. VM에서 SSH 상태 확인
```bash
systemctl status sshd
ss -tlnp | grep 22
```
 
2. Windows에서 ARP 테이블 확인
```cmd
arp -a | findstr 172.31
```
 
3. VM IP가 어느 인터페이스에 잡혔는지 확인
```
인터페이스: 172.31.0.1 --- 0x3   ← VMware 가상 어댑터 (잘못된 경우)
  172.31.0.254  00-50-56-...
 
인터페이스: 172.31.0.51 --- 0xa  ← 실제 물리 어댑터 (여기에 있어야 정상)
  172.31.0.254  08-00-27-...
```
 
4. 올바른 인터페이스로 라우팅 수동 추가 (Windows 관리자 cmd)
```cmd
route add 172.31.0.254 mask 255.255.255.255 172.31.0.1 if 10
route add 172.31.0.255 mask 255.255.255.255 172.31.0.1 if 10
```
 
> `if 10`은 인터페이스 번호(0xa = 10진수 10)이다. PC마다 다를 수 있으므로 arp -a에서 실제 사용 인터페이스 번호를 확인한다.
 
5. 영구 적용 (PC 재부팅 후에도 유지)
```cmd
route -p add 172.31.0.254 mask 255.255.255.255 172.31.0.1 if 10
route -p add 172.31.0.255 mask 255.255.255.255 172.31.0.1 if 10
```
 
**실무 적용 시 주의사항:**
 
- VMware, VPN, 가상 어댑터 등 여러 NIC가 있는 환경에서 자주 발생한다.
- `arp -a`로 VM IP가 어느 인터페이스에 잡혔는지 먼저 확인한다.
- `if` 번호는 환경마다 다르므로 반드시 확인 후 입력한다.
- `-p` 옵션 없이 추가하면 PC 재부팅 시 초기화되므로 영구 적용 시 반드시 `-p` 옵션을 사용한다.
