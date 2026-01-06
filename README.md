## [MySQL] Group Replication Communication Stack (33061 or 3306 포트)
: group_replication_local_address와 Communication Stack의 변화 (XCOM vs MySQL)

MySQL InnoDB Cluster를 구축하다 보면 네트워크 설정에서 혼란을 겪을 때가 있습니다. 분명 매뉴얼에는 Group Replication 통신을 위해 `33061` 포트를 열어야 한다고 되어 있는데, 막상 최신 버전(8.0 후반 ~ 8.4)으로 구축해 보면 `33061` 포트는 리슨(Listen)조차 하지 않고, `3306` 포트 하나만으로 클러스터가 잘 동작하는 현상을 목격하게 됩니다.

이번 글에서는 `group_replication_local_address` 변수의 의미와, MySQL 8.0.27 이전 버전에서 8.0.27 + 및 8.4에서의 **Communication Stack(통신 스택)**이 어떻게 변화했는지 정리해 봅니다.

## 1. group_replication_local_address 란?

`group_replication_local_address`는 Group Replication 멤버들이 서로 통신하기 위해 사용하는 **"각 멤버의 고유한 주소(IP:Port)"**를 정의하는 시스템 변수입니다.

클러스터 멤버들은 이 주소를 통해 다음과 같은 작업을 수행합니다.
* **Heartbeat:** 서로 살아있는지 생존 확인
* **Consensus:** 트랜잭션 처리에 대한 합의 (Paxos 알고리즘)
* **Certification:** 데이터 동기화 및 충돌 감지

즉, 클라이언트가 접속하는 포트(SQL Port)와는 별개로, **"DB 서버들끼리만 사용하는 내부 포트"** 입니다.

## 2. 8.0.27 버전 이전  vs 8.0.27 + & 8.4 : 무엇이 바뀌었나? (XCOM vs MySQL)

과거에는 이 변수에 `33061`과 같은 별도의 포트를 할당하는 것이 필수였습니다. 하지만 MySQL 8.0.27 이후, 그리고 8.4 LTS 버전에서는 **Communication Stack**의 변화로 인해 패러다임이 바뀌었습니다.

### A. 기존 방식: XCOM Stack (Legacy)
우리가 흔히 알고 있는 전통적인 방식입니다. 현재도 기본값은 이 값으로 설정되어 있습니다. 다만 mysql shell 을 이용해 클러스터를 생성 때 8.0.27 + 버전이면 기본값을 변경해줍니다.
* **프로토콜:** GCS(Group Communication System) 자체 프로토콜 사용
* **포트:** SQL 포트(3306)와 분리된 **전용 포트 필수** (권장: 33061)
* **보안:** IP 기반의 Allowlist (`group_replication_ip_allowlist`) 사용
* **특징:** 트래픽이 완전히 분리되지만, 방화벽 포트를 2개 관리해야 함.

### B. 최신 방식: MySQL Stack (Modern / 8.4 Default via Shell)
MySQL 8.0.27부터 도입된 새로운 방식입니다.
* **프로토콜:** 표준 **MySQL 프로토콜** 위에서 동작
* **포트:** **SQL 포트(3306)를 함께 사용** (별도 포트 불필요)
* **보안:** 사용자 계정 기반 인증 (SQL Grant 사용, Allowlist 무시)
* **특징:** 포트 하나만 열면 되므로 네트워크/방화벽 구성이 단순해짐.

## 3. 왜 내 서버는 'MySQL Stack'으로 설정되었을까?

가장 많은 오해는 *"나는 설정을 바꾼 적이 없는데?"* 라는 점입니다.
사실 MySQL Server(엔진) 자체의 기본값은 여전히 `XCOM`입니다. 하지만 클러스터를 구축할 때 사용하는 도구인 **MySQL Shell**의 동작 방식이 다릅니다.

MySQL Shell의 `dba.createCluster()` 메서드는 다음 로직을 따릅니다.

1. 대상 서버의 버전을 체크한다.
2. 버전이 **8.0.27 이상**인 경우, **`MYSQL` 통신 스택을 기본값(Default)**으로 선택한다.
3. `SET PERSIST` 명령을 통해 `mysqld-auto.cnf` 파일에 설정을 영구 저장한다.

즉, 사용자가 별도의 옵션(`communicationStack: 'XCOM'`)을 주지 않으면, 도구가 자동으로 최신 방식인 MySQL Stack을 적용해 버립니다. 이 때문에 사용자는 33061 포트를 설정한 적이 없음에도, 시스템은 3306 포트를 통해 통신하게 되는 것입니다.

## 4. 내 서버의 상태 확인하기

현재 내 클러스터가 어떤 방식을 사용 중인지 확인하려면 아래 쿼리를 실행하면 됩니다. 저는 8.4.5 버전을 설치했기때문에 MYSQL 방식으로 표시되는 것을 확인할 수 있습니ㅏ.
<img width="1938" height="384" alt="image" src="https://github.com/user-attachments/assets/14f0d195-4449-45ef-99f4-0d67bee5c41d" />

```sql
SELECT MEMBER_HOST, MEMBER_PORT, MEMBER_STATE, MEMBER_COMMUNICATION_STACK
FROM performance_schema.replication_group_members;

```
