# Airflow를 알아보자

Airflow는 $AIRFLOW_HOME(default는 ~/airflow)의 dags 폴더에 있는 dag file을 지속적으로 체크한다. 고로 보통 PYTHON으로 작성한 DAG파일들을 DAGS폴더안에 넣어 관리한다.

- DAG안에 여러개의 Task가 존재할수있다. 
- Task는 하나의 작업단위이다.
- Hook는 airflow외부 플랫폼과 데이터베이스를위한 인터페이스이다.
- airflow는 cron과 다르게 순서의 개념을가지고있다.
- 자신보다 앞에있는 순서의 작업이 실행되지않는다면 앞에있는 작업이 실행될때까지 기다린다.
- cron = 1 -> 2 에서 1이실패한다면, 지정한 주기마다 1부터다시시작
- airflow = 1 -> 2 에서 1이실패한다면 1의 작업이완료될때까지 2는기다린다.


<u>참고</u>

[airflow기초 개념](https://teki.tistory.com/6)

[airflow기초 문법](https://magpienote.tistory.com/194)


## GCP 환경에 Airflow를 설치하고 분산크롤링하기

초기엔 그저 여러사이트에서 기사를 수집하는 작업을 airflow를 통하여 진행하려고했으나, 여러사이트의 기사를 수집해오다보니 생각보다 시간이 꽤 오래걸리는점이 자꾸 마음에걸렸다.

그래서 GCP환경에 Airflow를 설치하고 여러대의 인스턴스에 크롤링작업을 분산시켜서 시간을 단축시키면 어떨까?라는 생각을 하게되었다.

그리하여 GCP환경에 Airflow를 통한 분산 크롤링작업을하는 과정을 기록으로 남기려한다.

총 3개의 인스턴스를 사용할것이고, 하둡과 마찬가지로 마스터-워커1-워커2의 구조를 가질것이다. 그리고 전처리를 통해 mysql과 하둡에 적재하는것까지 해볼것이다.


## GCP에 Airflow 설치

일단 airflow를 설치해야하니 airflow를 인스턴스에 설치할것인데 타 라이브러이와의 충돌방지를위해 가상환경에 설치하기를 권고하는듯하다. <u>그러므로 conda를 이용하여 가상환경을 만들어줄것이다.</u>

[아나콘다 공식 홈페이지](https://www.anaconda.com/download#downloads)

```bash

wget https://repo.anaconda.com/archive/Anaconda3-2023.07-2-Linux-x86_64.sh

GCP 인스턴스가 Ubuntu20 -x86을 사용중이니 위의 버전을 다운로드받았다.
.sh파일을 다운로드받았으면 권한을 변경해주고 실행한다

chmod 755 Anaconda3-2023.07-2-Linux-x86_64.sh

sh Anaconda3-2023.07-2-Linux-x86_64.sh or ./ Anaconda3-2023.07-2-Linux-x86_64.sh

실행하면 라이센스에대한 안내가뜨고, 엔터키를 눌러 마지막으로가면 동의 안내가 뜬다.

    Do you accept the license terms? [yes|no]
    >yes

이후 설치 경로에대해 안내가나온다. 엔터를 누른다.

    [/home/usr/anaconda3] >>>

그러면 설치경로대로 경로가 알아서 설정된다.


설치가 끝난후 bashrc에 들어가보면 아나콘다가 설치되면서 자동으로 등록된 환경변수가있다. 적용시켜주자.

source ~/.bashrc


적용이 끝났으면 가상환경을 하나 만들어주고 파이썬 버전을 지정해주자.

conda create -n [가상환경이름] python=[원하는버전]

나는 3.9버전을 사용할것이다.

conda create -n conda_airflow python=3.9 -y

가상환경을 활성화시켜주자

conda activate conda_airflow

# airflow
export PATH=$PATH:/home/계정명/.local/bin

or

export AIRFLOW_HOME=~/airflow (해당설정은 에어플로우에서 권장하는 기본값이다)

▲ 설치하기전에 경로를 지정해줬다.

이제 진짜 에어플로우를 설치해주자. 
단 에어플로우는 파이썬과 버전이안맞으면 오류를 일으키니, 버전에 맞는 에어플로우를 설치해줘야한다.

```
## (선택)GCP 인스턴스의 용량을 늘리고 디스크를 추가해서 사용해보자

설치하다가보니 GCP의 용량이 다 찬걸 확인했다.
생각보다 아나콘다가 정말 아나콘다급으로 매우 많은 용량을 차지하고있었다..

df -h 를 통해 디스크를확인해보니 used가 100%였다.
현재 작업을 진행하는 마스터인스턴스를 카피해서 새로만들자니 내부IP가 고정이아니여서, 새로운 인스턴스로 갈아타면 다시 설정해줘야하는 작업을해야하기때문에 귀찮았다..

그래서 영구디스크의 크기를 늘리고, 여유롭게 보조디스크를 하나 연결하는게 좋겠다는 생각을해서 사용중인 GCP인스턴스의 영구디스크 크기를 늘리는법과 새 디스크를 추가하였다.

[GCP영구디스크 크기 늘리기](https://cloud.google.com/compute/docs/disks/resize-persistent-disk?authuser=7&hl=ko&_ga=2.237255808.-1899439076.1677074651)

<mark>다만, 인스턴스의 영구디스크는 늘리는건되지만 줄이는건 안된다고한다.</mark>

이것을 사용하기위해 포맷을해줘야하니 포맷을 진행해주는데 포맷진행법은 리눅스와같다.

    $ sudo lsblk
    
    NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    loop0     7:0    0  55.7M  1 loop /snap/core18/2785
    loop n ....
    lopp n...

    sda       8:0    0    20G  0 disk
    ├─sda1    8:1    0  19.9G  0 part /
    ├─sda14   8:14   0     4M  0 part
    └─sda15   8:15   0   106M  0 part /boot/efi
    sdb       8:16   0    10G  0 disk
    
    위의 명령어를 입력하면 디스크들이나오는데 저기 맨아래의 sdb가 새로추가한 디스크다.
    아래의 명령어를 차례로 입력해주자
    
    #HDD를 EXT4로 포맷
    sudo mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb

    # 마운트 포인트 생성

    sudo mkdir -p /mnt/disks/data

    #HDD 마운트
    sudo mount -o discard,defaults /dev/sdb /mnt/disks/data


    #HDD 권한추가

    sudo chmod a+w /mnt/disks/data

    # 마운트 확인

    df -H | grep /mnt/disks/data
    /dev/sdb         11G   25k   11G   1% /mnt/disks/data



### VM이 다시 시작할때 자동마운트하게만들자

    # fstab 백업파일 만들기
    
    sudo cp /etc/fstab /etc/fstab.backup

    # /etc/fstab에 기입할 UUID확인

    sudo blkid /dev/sdb

    그러면 UUID가 UUID="a0alskd1-31sdk-..." TYPE="ext4" 와 같은 포맷으로나오는데
    ""안의 UUID의 VALUE값만 복사.

    # /etc/fstab를 열어 맨 아래에 아래 uuid와 아래내용 붙여넣기

    sudo vi /etc/fstab

    UUID=UUID_VALUE(아까복사한값) /mnt/disks/data ext4 discard,defaults,nofail 0 2​

    # 재부팅후에도 제대로 마운트를하고있는지 /dev/sdb 마운트 확인

    sudo df -h

    Filesystem      Size  Used Avail Use% Mounted on
    /dev/root        20G   11G  8.5G  57% /
    devtmpfs        3.9G     0  3.9G   0% /dev
    tmpfs           3.9G     0  3.9G   0% /dev/shm
    tmpfs           794M 1004K  793M   1% /run
    tmpfs           5.0M     0  5.0M   0% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/loop0       56M   56M     0 100% /snap/core18/2785
    loop n ...         ..  ..     ..    . ...
    ...
    /dev/sdb        9.8G   24K  9.8G   1% /mnt/disks/data

    위와같이 재부팅해도 디스크가 잘연결되어있는걸 볼수있다.


참고

[gcp디스크 추가](https://yooloo.tistory.com/m/156)

```bash

이제 다시 에어플로우를 설치해주자

pip install "apache-airflow[celery,redis,kafka,mysql,spark,elasticsearch,hdfs]==2.7" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.7/constraints-3.9.txt"

-> airflow버전은 2.2.3, 파이썬버전은 3.9라는것을 의미한다. 

※[celery]는 airflow 분산처리를위해 워커역할을하는 비동기 작업큐이다. 주로 레디스나 레빗mq와 함께쓴다. airflow celery worker 명령어로 워커 인스턴스를 붙여주면 된다.

※레빗mq / 카프카

-카프카는 소비자가 메세지의 소비여부와 상관없이 메세지를 계속 게시하고,
-레빗mq는 해당 메세지를 보낼 특정 소비자가 메세지를 소비하는지 지속적으로 모니터링한다.

※레빗mq / 레디스

-레빗mq는 크고 복잡한 작업,메세지에 더 적합하고 메세지브로커로써 레디스보다 더 많은 기능을 제공한다.

-레디스는 Key-Value를 이용해 Celery가 처리할 작업을 Celery에 보낸 후 Cache 에서 해당 Key를 제거하는 방식으로 작동한다. 지속성이중요하지않고,약간의 손실을 견딜수있는 짧은 보존메세지에 적합하다. 대신 메모리에서 캐시를 가져다쓰기때문에 처리속도가빠르고 큰 메세지를 처리할때는 대기시간이 오래걸린다.

에어플로우가 별일없이 잘 설치되었으면 db초기화를 해준다.

#db초기화
airflow db init

db를 초기화시켜주면 지정한 경로에 airflow 디렉터리가 생긴다.
airflow의 기본 db는 sqlLite이다. sqlLite는 분산처리에 적합하지않으므로 Mysql이나 Postgresql을 사용해야한다.

하지만 일단 에어플로우가 정상작동하는지 먼저 테스트를해보고 db를변경해줄것이다.
(왜냐면 나중에 오류나면 어디서부터 오류가났는지 모르니까..)


#관리자 정보 생성

airflow users create --username admin --password "비밀번호" --firstname 이름 --lastname 이름 --role Admin --email 관리자메일주소

생성이 되면 아래처럼 

User "admin" created with role "Admin"

라고나온다. --username 부분이 아이디라고 생각하면된다.

# 실행해보자

airflow webserver -p 8080

-p옵션은 포트지정이다. 기본 8080으로 지정되어있다.
GCP환경이니 airflow가 동작되고있는 인스턴스의 [외부ip:포트]를 브라우저에 입력해서 접속하면된다.

# 같은 인스턴스를 다른 쉘에서 접속해서 스케쥴러도 실행해준다.

airflow scheduler


```

![airflow최초 로그인](./airflow/초기login.PNG)

브라우저에 위와같은창이 뜨면 성공

에어플로우를 종료가 잘안된다.
그럴땐 8080, 즉 아까 airflow 웹서버를 띄운 포트를 점유하고있는 프로세스들을 찾아 모두 강제로 없애주면된다.

```sudo lsof -i tcp:8080```

```sudo lsof -i tcp:8080 | grep 'user' | awk '{print $2}' | xargs kill -9```


기본적으로 실행이 잘되었으니 이제 DB를 바꿔주고 자잘한 설정들을 바꿔준다.

계정에 관련한 기본 명령어는 아래사이트에서 참고했다.

[Airflow_계정관련](https://www.bearpooh.com/150)

각종 설정과 DB를 변경해주기전에 방금생성한 테스트계정이 마음에들지않으니 삭제하고 재생성해줬다.

```airflow delete [-u username or -e email ]```

<mark>user name이나 email을 지정해주는것만으로도 삭제가 가능한데, 주의할것은 경고메세지를 띄우지않으니 계정을 삭제할때는 꼭 확인을하자.</mark>

**./airflow.cfg 파일을 편집해주자**

```bash

(airflow디렉토리 바로아래 있음)

vim airflow.cfg

#18번째

default_timezone = utc
->default_timezone = Asia/Seoul

#24번째
executor = LocalExecutor #excutor = airflow task 실행 메커니즘이다. 여기에 어떤방식을 적용하느냐에따라 task의실행 방식이 달라진다. 

## ※기본값 Sequential = Airflow에서 제공하는 기본 Executor로 sqlite와 함께 사용할 수 있는 Executor로 한번에 하나의 task만 실행할 수 있어 병렬성을 제공하지 않아 실제 운영환경에는 적합하지 않음

## Local = task를 병렬로 실행하는 것이 가능하며, 옵션값을 통해 최대 몇 개의 task를 병렬로 실행할지 설정하는 것이 가능하다. self.parallelism 옵션 값을 이용해 설정하며 이 설정값을 0으로 설정하는 경우 Local Executor는 task를 제한없이 무제한으로 실행하게 되며 이를 Unlimited parallelism 이라고 한다.

## Celery = Celery Executor 역시 Local Executor과 마찬가지로 task를 병렬로 실행할 수 있다. Celery는 추가적으로 Redis나 RabbitMQ같은 MQ를 추가적으로 필요로 하는데 이는 Celery Executor가 클러스터 형식으로 구성되고 MQ에 있는 task를 실행하는 구조로 동작하기 때문이다. 따라서 Celery Executor 클러스터 형식으로 구성할 수 있어 Executor에 대한 HA 구성과 Scale out이 자연스럽게 가능하며 LocalExecutor보다 실제 운영환경에 적합하다고 판단된다. 다만 DAG 파일 역시 Celery Executor로 사용하고 있는 모든 Worker에 배포되어야 하기 때문에 git을 이용해 DAG를 관리하고 배포하는 시스템을 구축해야 한다. 

## Celery = worker n개 / HA 가능 / task 병렬처리 가능
## Local = worker 1개 / HA 불가능 / task 병렬처리 가능

나는 Celery로 분산크롤링,처리 할거기때문에 Celery를 선택했다.


#52번째

load_examples = True
->에어플로우가 만들어놓은 예제 노출여부 안볼거니 load_examples = False


#206번째

sql_alchemy_conn = sqlite:////home/리눅스계정명/airflow/airflow.db
->sql_alchemy_conn = mysql+mysqldb://airflow:비밀번호@localhost:3306/airflow 
mysql의 접속 정보이다. airflow는 sql 데이터베이스명, localhost:3306은 접속경로
접속하려는 db정보가 다른경우 airflow부분 대체가능

2.7버전일경우 아래 참조

(https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html)

#428번째

endpoint_url = http://localhost:8080
-> http기본경로가 8080이기때문에 바꿔준다. 끝을 8988로

#525번째
base_url = http://localhost:8080
->base_url = http://localhost:8988

#531번째
default_ui_timezone = UTC
->default_ui_timezone=Asia/Seoul

#537번째
web_server_port = 8080
->web_server_port = 8988

#963번째

dag_dir_list_interval = 300
->dag라는 파일의 새로고침 시간 기본값 5분씩. 연습할때는 자주 새로고침해놓고, 프로젝트땐 길게잡아놔도된다.

-> dag_dir_list_interval = 60 으로 변경해주자

#1333번째
dag_default_view = grid
-> dag_default_view = graph

airflow가 버전이 업데이트되면서 WEBUI의 기본 VIEW가 grid로 설정되어있다. 나는 그래프로보고싶으니 그래프로 바꿔준다.

```

이제 에어플로우의 기본적인 설정을 바꿔줬으니 sqlLite를 사용하고있는 airflow db를 앞으로 Mysql을 쓰라고 알려주기위해 <u>초기화해줘야한다.</u>
하지만 db를 초기화해주기전에 필요한 패키지를 먼저 설치해야한다.


<Mysql과 관련된 패키지>

    sudo apt install libmysqlclient-dev -y

    sudo apt install default-libmysqlclient-dev pkg-config -y

    pip install apache-airflow-providers-mysql

    pip install mysqlclient 

    ★pip install mysql-connector-python

<그 외 airflow에서 외부서비스를 사용하기위해 설치한 패키지>

    pip install apache-airflow-providers-apache-hdfs
        -> gcp의 인증문제인가? 연관패키지모두 설치가안되어서  conda install -c conda-forge apache-airflow-providers-apache-hdfs 로 설치하였다.

    pip install apache-airflow-providers-apache-kafka
    pip install apache-airflow-providers-apache-spark
    pip install apache-airflow-providers-elasticsearch


필요한 패키지를 모두 설치해줬으면 이제 DB를 초기화시켜주자

```airflow db init ```

db를 초기화시켜주니 정상적으로 초기화가되고 작동은하지만 

WARNING: MYSQL_OPT_RECONNECT is deprecated and will be removed in a future version.
오류가 계속뜬다. 그래서 아래와같은 패키지를 설치하였다.

```sudo apt install libdbd-mysql-perl```

하지만 패키지를 설치한다고 에러가 사라지지않았다.
또한 같은오류가 에러 로그에 너무많이찍히고 dag파일도 제대로 실행되지않았다.
찾아보니 삭제될 예정의 기능인데 어떤 라이브러리가 mysql의 RECONNECT를 자꾸 호출하여 경고메세지가 발생하는것같다. 문제는, 어떤 라이브러리가 요청을 보내는지 알수없기때문에 답답했다..

그러다가 공식문서를찾아보고 정보를 수집해보니 결국 <mark>mysqlclient 라이브러리가 문제였음을 발견했다.</mark>

하지만 해당 라이브러리를 삭제하면 영향받는 라이브러리가많기때문에, airflow와 mysql를 이어주는 커넥터를 바꾸기로했다.

```pip install mysql-connector-python```

<u>spark에서 mysql을 사용하기위해 설치했던 커넥터다.</u>
나는 airflow와 스파크가 같은 인스턴스내에 있기때문에, 미리 설치해놨지만 만약 mysqlclient를 통해 airflow와 mysql을 연결했을때 나와같은 오류가난다면 저 라이브러리를 사용하는것을 추천한다.

그럼 airflow의 sql설정을 바꿔주고 db를 초기화시켜주자.


    sql_alchemy_conn = mysql+mysqldb://<user>:<password>@<ip><port>/<dbname>
                                    ↓↓↓↓↓↓↓↓↓↓↓↓↓
    sql_alchemy_conn = mysql+mysqlconnector://<user>:<password>@<ip>:<port>/<db_name>

    airflow db init (※2.7버전이면 경고메세지뜸. airflow db migrate 로 해야한다.)

    Initialization done.

db를 초기화했으면 관리자 계정도 날아갔을것이므로 관리자계정도 새로만들어주자.
간단한 DAG파일을 만들어서 AIRFLOW를 통하여 DB에 데이터가 적재되는지 확인해보자.

먼저 파이썬dag파일을 만들기전에 해야할 사전작업이있다.
connection 정보를 입력해줘야한다.

1. airflow web을 실행시켜 web 접속

[airflow_web_connection1](./airflow/web_connections_1.PNG)


connection을 눌러 새로운 접속을 하나 추가해준다.

2. 저장할 db정보 입력

[airflow_web_connection2](./airflow/web_connections_2.PNG)


- Connection Id : 여기에 입력한 값과 dag파일의 conn_id의 값이 일치해야한다. 연결정보의 고유 id같은것 
- Connection Type : 말그대로 연결할 타입. 목록에서 원하는 DBMS 골라주면된다
- host : 연결할 db가있는 인스턴스의 IP. (GCP의경우 내부IP로입력)
- Schema : 작업할 DB의 이름
- id : DBMS에 등록된 ID
- password : id의 password
- port : 사용하려는 DBMS가 점유하고있는 포트번호

기본적으로 작성해야할 정보들은 위와같으며, 모두 작성하였으면 저장해주고 Task와 DAG파일을 작성해주면된다.


```python

#mysqlHook를 사용해도 무관.
#airflow 버전은 2.7이므로 버전에따라 적절히 바꿔사용할것

from datetime import datetime, timedelta
from email.policy import default
from textwrap import dedent
from airflow import DAG
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

default_args = { #보통 누가만들었는지, START_DATE는 언제인지 정의함.생성에관한정보
    'owner' : 'gcp_master' #DAG를 누가만들었는지, 생성자가누구인지 알리는것
    'depends_on_past': False, #과거에 실패한 task인스턴스들을 실행할것인지 여부
    'retires': 1, # task가 실패했을때 재시도할 횟수
    'retry_delay': timedelta(minutes=5),
    'conn_id' : 'mysql_test'
}

sql_create_table = """
    CREATE TABLE `employees` (
        `employeeNumber` int(11) NOT NULL,
        `lastName` varchar(50) NOT NULL,
        `firstName` varchar(50) NOT NULL,
        `extension` varchar(10) NOT NULL,
        `email` varchar(100) NOT NULL,
        `officeCode` varchar(10) NOT NULL,
        `reportsTo` int(11) DEFAULT NULL,
        `jobTitle` varchar(50) NOT NULL,
    PRIMARY KEY (`employeeNumber`)
    );
"""

sql_insert_data = """
    insert  into `employees`(`employeeNumber`,`lastName`,`firstName`,`extension`,`email`,`officeCode`,`reportsTo`,`jobTitle`) values 
        (1002,'Murphy','Diane','x5800','dmurphy@classicmodelcars.com','1',NULL,'President'),
        (1056,'Patterson','Mary','x4611','mpatterso@classicmodelcars.com','1',1002,'VP Sales'),
        (1076,'Firrelli','Jeff','x9273','jfirrelli@classicmodelcars.com','1',1002,'VP Marketing'),
        (1088,'Patterson','William','x4871','wpatterson@classicmodelcars.com','6',1056,'Sales Manager (APAC)'),
        (1102,'Bondur','Gerard','x5408','gbondur@classicmodelcars.com','4',1056,'Sale Manager (EMEA)'),
        (1143,'Bow','Anthony','x5428','abow@classicmodelcars.com','1',1056,'Sales Manager (NA)'),
        (1165,'Jennings','Leslie','x3291','ljennings@classicmodelcars.com','1',1143,'Sales Rep'),
        (1166,'Thompson','Leslie','x4065','lthompson@classicmodelcars.com','1',1143,'Sales Rep'),
        (1188,'Firrelli','Julie','x2173','jfirrelli@classicmodelcars.com','2',1143,'Sales Rep'),
        (1216,'Patterson','Steve','x4334','spatterson@classicmodelcars.com','2',1143,'Sales Rep'),
        (1286,'Tseng','Foon Yue','x2248','ftseng@classicmodelcars.com','3',1143,'Sales Rep'),
        (1323,'Vanauf','George','x4102','gvanauf@classicmodelcars.com','3',1143,'Sales Rep'),
        (1337,'Bondur','Loui','x6493','lbondur@classicmodelcars.com','4',1102,'Sales Rep'),
        (1370,'Hernandez','Gerard','x2028','ghernande@classicmodelcars.com','4',1102,'Sales Rep'),
        (1401,'Castillo','Pamela','x2759','pcastillo@classicmodelcars.com','4',1102,'Sales Rep'),
        (1501,'Bott','Larry','x2311','lbott@classicmodelcars.com','7',1102,'Sales Rep'),
        (1504,'Jones','Barry','x102','bjones@classicmodelcars.com','7',1102,'Sales Rep'),
        (1611,'Fixter','Andy','x101','afixter@classicmodelcars.com','6',1088,'Sales Rep'),
        (1612,'Marsh','Peter','x102','pmarsh@classicmodelcars.com','6',1088,'Sales Rep'),
        (1619,'King','Tom','x103','tking@classicmodelcars.com','6',1088,'Sales Rep'),
        (1621,'Nishi','Mami','x101','mnishi@classicmodelcars.com','5',1056,'Sales Rep'),
        (1625,'Kato','Yoshimi','x102','ykato@classicmodelcars.com','5',1621,'Sales Rep'),
        (1702,'Gerard','Martin','x2312','mgerard@classicmodelcars.com','4',1102,'Sales Rep');
"""


with DAG( #데코레이터를 import시켜 @dag,@task 와 같이 사용가능
    'connect_to_local_mysql', #dag id
    default_args = default_args,
    description = #description=동작내용 묘사 정의. 없어도 무관. 메모같은기능
    """ 
        1) create 'employees' table in local mysqld
        2) insert data to 'employees' table
    """,
    schedule = '@once',
    start_date = datetime(2023, 8, 20),
    catchup = False,
    tags = ['mysql', 'local', 'test', 'employees']
) as dag:
    t1 = SQLExecuteQueryOperator( #sqlexceutequertoperator안에있는 인자를써야함
        task_id="create_employees_table",
        sql=sql_create_table #sql문삽입. sql에작동할내용
    )

    t2 = SQLExecuteQueryOperator(
        task_id="insert_employees_data",
        sql=sql_insert_data
    )


    t1 >> t2

```

이로써 마스터1, 워커 1 , 워커 2 에 모두 airflow를 설치하였고 세개의 인스턴스 모두 같은 DB를 사용하게 설정하였다.

그전에 현재 mysql에 연결되어있는 airflow의 계정은 airflow의 정보들을 관리하기위해 airflow 전용 DB에만 접근/작업 권한이있기때문에 airflow 계정에 크롤링한 데이터를 적재할 DB에 접근할수있는 권한을 추가해준다.

또한 아주당연한거지만 워커들이 master의 mysql에 접속할수있어야하기때문에, 워커인스턴스에도 mysql이 설치되어있어야한다.

```sql

grant ALL PRIVILEGES ON 'your_db'.* to 'airflowid'@'%';

flush privileges;

```

이제 마스터와 워커들을 연결해주기위해 celery를 설치하고 노드들을 연결하는 설정을해주자.


## celery,Redis 설치 및 Airflow worker노드연결

```bash

설치
    [마스터,워커1,워커2 공통]
    pip install Redis

    pip install celery

    #제대로 설치가되었는지 확인용으로

    celery --version

    #redis는 직접다운로드받는방식과 아래처럼 apt를 이용하는방법이있다. 나는 apt를 이용하여 redis-server를 설치한다.

    sudo apt-get install redis-server -y

    #redis를 apt로 설치를진행할시, 실행까지 진행된다. 그러니 설치가끝나고 실행되고있는지 확인해보자

    sudo service redis status
    (Active: active (running)) 확인

    netstat -an |grep 6379

    tcp        0      0 127.0.0.1:6379          0.0.0.0:*               LISTEN
    tcp6       0      0 ::1:6379                :::*                    LISTEN

    이것은 reids의 기본설정이다. 기본적으로 redis의 설정파일에는 
    ipv4 : 127.0.0.1 / ipv6 : ::1로, 로컬시스템내에 또다른 클라이언트-서버 관계를 구성해놓은 루프백으로 되어있다.
    그러므로 다른 서버에서 redis에 접근하거나 airflow에서 localhost외에 접근하게하고싶으면 conf파일을 찾아 bind 부분에 ip를 설정해주면된다.

    redis의 실행을 확인했으면 테스트를해보자

    cd /usr/bin # redis가 설치되어있는곳.  redis-benchmark/redis-check-aof/redis-check-rdb/redis-cli/redis-server 등이 있다.
    sudo redis-cli #cli로 실행
    
    
    127.0.0.1:6379> #redis를 실행해보면 기본포트는 6379번으로 지정되어있고 아무런반응이없다.

    #ping을입력하여 redis가 실행되고있는지 확인한다. ping을 입력하면 PONG이라고 메세지가온다.
    127.0.0.1:6379> ping 
    PONG
    
    #SET/GET을 이용하여 메세지를 송/수신해보자

    127.0.0.1:6379> set bluedeveloper gcp_redis_hi
    OK #SET으로 데이터를보내면 대답이돌아옴

    127.0.0.1:6379> get bluedeveloper
    "gcp_redis_hi" #GET으로 Redis에 보낸 메세지를 받아올수있다.

    redis의 ip설정을 바꿔주자. conf파일은 아래와 같은 경로에있다.

    /etc/redis/redis.conf

    # 허용가능한 ip를 바꿔주자
    # [master필수, 워커1,2 선택.] broker를 master로뒀기때문에 나는 워커1,2의 설정은 건드리지않았다.

    #bind 127.0.0.1 ::1 (기본)
    #bind 0.0.0.0 (모든외부접속허용)
    bind host_name1 host_name2 host_name3 ...(,로 구분하지않고 공백으로 구분. 저장한 갯수만큼의 호스트에서 접속가능)

    # 접속오류가나지않는지 확인

    airflow celery flower

    flower의 경우 원래 worker들을 먼저 실행해야 경고가뜨지않는다.
    하지만 우리는 redis접속오류가 나지않는지 확인할것이기때문에 경고를무시하고 오류가안뜨는지 확인한다.

기본적인 동작 테스트는 끝났으니 이제 airflow의 cfg파일을 수정하고 celery로 인스턴스를 이어주자.

    vi airflow.cfg

    # redis 설정

    broker_url = redis://host_name:port

    스케줄러와 워커가 서로 통신할 redis의 호스트명과 reids의 포트명을 작성해주면된다.

    result_backend = db+mysql+mysqlconnector://계정명:비밀번호@host_name:port/db명

    워커들의 결과를 저장할 db를지정하는 부분이다. 나는 sql_alchemy_conn에서 사용한 airflow전용 db에 결과를 저장할것이므로, sql_alchemy_conn과 같은 값을 사용하였다.

    # flower port 설정

    flower_port = 5555 -> flower_port = 15555
    워커들에도 모두 airflow.cfg파일을 수정해줬으면 실행해본다.

    # node1(master)는 스케줄러+웹서버+flower

    airflow webserver -D
    airflow scheduler -D
    airflow celery flower -D

    # [워커1, 2에서]
    ## -q = 워커가 소비할 큐 이름 지정. 지정하지않을시 디폴트로 간주 / -D = 데몬으로실행

    airflow celery worker -q q_name1 -D 
    airflow celery worker -q q_name2 -D

    ※기본적으로 airflow의 큐관련 설정을 건드리지않았다면, 스케줄러는 디폴트 라는 이름을 가진 큐에 작업명령을내리게된다.
    고로, 큐를 따로 설정하지않은상태로 worker의 큐 이름을 q_name1이라고 지정하게된다면 redis에는 q_name1이라는 키가 생성되게되고, 이 키 안에 스케줄러가 작업을 배치할때까지 일을안하고 놀게된다..(나도 알고싶지않았음)
    그렇게되면 스케줄러는 디폴트큐에 작업을 쌓게되고 쌓인 작업은 워커들이 소비하지않기때문에 계속쌓이게되고 대기상태에빠진다.
    
    이 큐 설정에대해서는 아래에서 조금 더 알아본다.


위와같이 실행하고 워커들이 잘붙는지 flower에 들어가서 확인해본다.

```

![flower_web_ui](./airflow/airflow_celery.PNG)

워커들이 잘 붙은것을 볼수있다.

이제 정상적으로 작동하는지 확인하기위해 위에서 간단히 Mysql에 테이블을 생성하는 워크플로우를 약간변형하여 병렬처리가되는지 확인해본다.


```python




```



