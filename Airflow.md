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

#1868번째 (선택)
catchup_by_default = True
->False
#1143번째 (선택)
default_queue = default
->default_queue = airflow-master

airflow가 버전이 업데이트되면서 WEBUI의 기본 VIEW가 grid로 설정되어있다. 나는 그래프로보고싶으니 그래프로 바꿔준다.

또한 기본적으로 큐 지정을안하면 redis의 default 큐로보내버리니, 기본큐를 airflow-master라고 바꿔주었다.


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

    #bind 127.0.0.1 ::1 (기본) ::1은 ipv6
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

    result_backend = db+mysql://계정명:비밀번호@host_name:port/db명

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

import pendulum
from datetime import datetime, timedelta
from email.policy import default
from textwrap import dedent
from airflow import DAG
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

kst = pendulum.timezone("Asia/Seoul")

default_args = { 
    'owner' : 'gcp_master', 
    'depends_on_past': False,
    'retires': 1,
    'retry_delay': timedelta(minutes=5),
    'conn_id' : 'mysql_test'
}

create_table = """
    CREATE TABLE `reduce_table` (
        `id` int(10) NOT NULL,
        `name` varchar(50) NOT NULL,
        `press` varchar(50) NOT NULL,
    PRIMARY KEY (`id`)
    );
"""

insert_data = """
    insert  into `reduce_table`(`id`,`name`,`press`) values 
        (101,'hadoop','apachehadoop'),
        (102,'spark','apachespark'),
        (103,'airflow','apacheairflow');
"""

add_data = """
    insert  into `reduce_table`(`id`,`name`,`press`) values 
        (104,'elastic','elk'),
        (105,'kafka','apachekafka'),
        (106,'zookeeper','apachezookeeper');
"""

with DAG(
    'airflow_reduce_test',
    default_args = default_args,
    schedule = '@once',
    start_date = datetime(2023, 9, 3, tzinfo=kst),
    catchup = False,
    tags = ['test','reduce','celery'],
    description = """
        1) t1 = create 'reduce_table' table in local mysqld
        2,3_병렬처리) t2 = insert data to 'reduce_table' table 'id = 101~103' / t3 = insert data to 'reduce_table' table 'id = 104~106'
    """

)as dag:
    t1 = SQLExecuteQueryOperator(
        task_id="airflow_reduce_create_table",
        sql=create_table
    )

    t2 = SQLExecuteQueryOperator(
        task_id="airflow_reduce_insert_data",
        sql=insert_data
    )

    t3 = SQLExecuteQueryOperator(
        task_id = "airflow_reduce_add_data",
        sql=add_data
    )

t1 >> [t2,t3]

```



![간단한병렬처리](./airflow/airflow_reduce.PNG)

<mark>airflow의 web UI로 보면 Task의 흐름은 그래프와같다.</mark>

conn에 등록해놓은 id의 데이터베이스에 t1에 작성한대로 테이블이 생겼는지, 
t1 = 101~103 /  t2 = 104~106 의 데이터가 들어가져있는지 확인해본다.

![간단한병렬처리결과](./airflow/airflow_mysql_reduce.PNG)

<u>테이블과 데이터들이 잘 들어간것을 확인할수있다.</u>

하지만 병렬처리를 수행할때 중요한 부분이있다.
**바로 마스터인스턴스(web,scheduler만실행하는 인스턴스)에만 dag파일을 배포하면안된다는것이다.**
한곳에만 dag파일을 업로드한다면 오류가발생하는것을 확인하였다.

<mark>airflow를 구성하고 있는 모든서버의 동일한 위치, 동일한 이름으로 dag 파일이있어야지 실행된다.</mark>


그럼 이제 큐를 지정하는방법을 알아보자. 큐를 이용하는 방법은 간단하다.
그냥 DAG파일을 작성할때 해당 TASK를 처리할 큐의이름을 지정해주면된다.

만약 worker1과worker2 인스턴스를 각각 q_name1, q_name2라는 이름을 주고 실행시킨다고 가정한다.

```worker1 > airflow celery worker -q q_name1```

```worekr2 > airflow celery worker -q q_name2```

그리고 위의 DAG파일 예제를 이용해 큐를 지정해본다.

```python
....
....

with DAG(
    'airflow_reduce_test',
    default_args = default_args,
    schedule = '@once',
    start_date = datetime(2023, 9, 3, tzinfo=kst),
    catchup = False,
    tags = ['test','reduce','celery'],
    description = """
        1) t1 = create 'reduce_table' table in local mysqld
        2,3_병렬처리) t2 = insert data to 'reduce_table' table 'id = 101~103' / t3 = insert data to 'reduce_table' table 'id = 104~106'
    """

)as dag:
    t1 = SQLExecuteQueryOperator(
        task_id="airflow_reduce_create_table",
        sql=create_table,
        queue="q_name1" # t1을 처리할 큐 이름. airflow.cfg파일의 브로커아이디에 작성한 redis의 q_name1이라는 곳에 작업배치
    )

    t2 = SQLExecuteQueryOperator(
        task_id="airflow_reduce_insert_data",
        sql=insert_data,
        queue="q_name1"
    )

    t3 = SQLExecuteQueryOperator(
        task_id = "airflow_reduce_add_data",
        sql=add_data,
        queue="q_name2" #t2을 처리할 큐 이름. airflow.cfg파일의 브로커아이디에 작성한 redis의 q_name2이라는 곳에 작업배치
    )

t1 >> [t2,t3]

```

위와같이 큐 이름을 지정해주면 스케줄러가 airflow.cfg파일안에 작성한 브로커url를 따라 그곳의 큐 이름과 일치하는곳에 작업을 배치한다.
그리고 워커들은 쌓인 작업을 처리한다.

Airflow가 분산처리를 하는것을 확인했으니, 이제 본 글의 목표인 airflow+spark+python를 이용한 크롤링 데이터파이프라인을 구축해볼것이다.

spark와 크롤링하는부분을 하나씩 설정해보자.

앞서, 나의 GCP구성은

<u>현재 airflow의 마스터노드역할을하는 인스턴스는 spark의 마스터노드 역할도 겸임한다. 또한, 이곳에는 elasticsearch와 airflow의 메타데이터역할을하는 mysql도 함께 운영되고있다.</u>

나는 Spark,Hadoop을 이미 마스터 노드,워커 노드에 모두 설치하여 분산환경을 구축해줬으므로 설치에관해서는 건너뛰겠다.

이제, 크롤링에 필요한 드라이버와 패키지를 다운로드받아주고 간단한 크롤링, 그리고 전처리를 테스팅해보고 프로젝트에 필요한 사이트를 일정주기마다 크롤링하는 코드를 작성해보겠다.


나는 크롤링을하기위해 **Selenium**과 **Firefox**, **geckodriver**, spark를 동작을 쉽게보기위해 재플린노트북을 설치할것이다.

사실 재플린노트북같은경우는 airflow로 스파크를작동시키기전 코드에 이상이있는지, 데이터에 이상이있는지 확인하기위해 설치하는것이다.
하지만 spark를 실행시켜 spark-submit을 통해 동작확인을해도되지만, 조금 더 가독성이 좋다는 부분에서 재플린을 사용하기로했다.


```python

마스터를 철저히 분리시키고싶다면 크롤링도구들을 설치하지않아도되지만, 나는 마스터에도 일단 설치해주겠다.

[마스터, 워커 공통 = Selenium,Firefox,geckodriver / 마스터 = spark 작업확인용 재플린노트북]

#나는 GUI를 사용하지않는 GCP를사용하고있으니 터미널로 Fierfox를 다운로드받아줄것이다.

sudo add-apt-repository ppa:ubuntu-mozilla-security/ppa

sudo apt-get update

sudo apt-get install firefox -y

#firefox 버전확인

firefox -v

Mozilla Firefox 117.0

위와같이 나오면 성공.


#selenium설치
## wget을 이용하여 수동으로 설치하는법이있고 pip 를 이용하여 설치하는방법도있다. airflow를 이용하여 spark,크롤링을할것이므로 conda명령어를 이용하여 설치하였다.

conda install -c conda-forge selenium -y


#fierfox용 웹드라이버 geckodriver 설치

https://github.com/mozilla/geckodriver/releases

위의 링크에서 각 OS에맞는 패키지의 링크를 복사해와서 wget으로 다운로드받아준다.

tar.gz이므로, 압축을해제한다

tar xvzf geckodriver-v0.33.0-linux64.tar.gz

위의 패키지들을 모두 받아줬으면 크롤링을 실행해보자.
```

### 간단한 크롤링 테스트

```python
#Daily Gaewon
#반려동물 상식을 반려동물기사 사이트에서 크롤링


from selenium import webdriver
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import FirefoxOptions
from selenium.common.exceptions import NoSuchElementException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from pyspark.sql import *
from pyspark.sql.functions import regexp_replace
from pyspark.sql.functions import col
import re
import time

#시간측정을위해 사용.

start_time=time.time()

#web드라이버 지정. 셀레니움 4버전대부터는 셀레니움이 알아서 웹드라이버를 불러오기때문에 사용하지않는 옵션이다.

webdriver_service = Service('/usr/bin/geckodriver') #(윈도우)경로를 \ 로 복사하지말고, /로변경해줄것.

opts = FirefoxOptions()
opts.add_argument("--headless")
# opts.binary_location = r'C:\Program Files\Mozilla Firefox\firefox.exe'#(윈도우)에러,moz:firefoxOptions.binary를 해결하기위해, 파이어폭스 경로를 수동지정.

#browser = webdriver.Firefox(service=webdriver_service, options=opts) -> 셀레니움이 최신버전으로 업데이트되면서 드라이버경로지정이필요없어졌다.

browser = webdriver.Firefox(options=opts)

# 한 페이지내에서 모든 뉴스의 갯수만큼 타이틀 / 링크를 따로추출하여 두개의 리스트에 저장.
dailygaewon_list=[] #뉴스정보를 저장할 리스트
dailygawon_link=[] #본문링크만 저장할 리스트(본문크롤링에 필요하기에 따로 저장)

#페이지를 순환할 for문 / 데일리개원에 미디어 페이지를 크롤링해온다.
for pages in range(1,7): #6페이지까지니까 +1해서 7 / 또한 원래 페이지의 URL이 page={n}&total=nnn&sc_section_code=~이지만,total은 늘 변하는값이므로, 빼줬다.빼줘도 작동지장x
    base_url = f"http://www.dailygaewon.com/news/articleList.html?page={pages}&sc_section_code=&sc_sub_section_code=S2N30&sc_serial_code=&sc_area=&sc_level=&sc_article_type=&sc_view_level=&sc_sdate=&sc_edate=&sc_serial_number=&sc_word=&box_idxno=&sc_multi_code=&sc_is_image=&sc_is_movie=&sc_order_by=E"
    browser.get(base_url)

    #페이지 내의 모든 기사의 갯수를추출하여 변수로 저장
    all_news=browser.find_elements(By.CLASS_NAME,"table-row")
    all_news_num=len(all_news)
    
    #기사의 길이만큼 for문을 반복해서 타이틀, 링크를 따로 추출
    for all_news_n in range(1,all_news_num+1): #1~19까지만 작동되니까 +1을해서 갯수를맞춰줌
        news_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a/strong").text
        news_link=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a").get_attribute("href")
        dailygaewon_list.append([news_title,news_link])
        dailygawon_link.append(news_link) #이것은 데이터 프레임으로 만들지 x
# print(dailygaewon_list)
# print(news_link)


#링크도 데이터프레임에 추가했음.
dailygawon_contents=[]
for dailygawon_links in dailygawon_link:
    browser.get(dailygawon_links)
    news_cont_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/div/div").text
    news_cont=browser.find_element(By.CSS_SELECTOR,"div#article-view-content-div").text
    news_date=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/section/div/ul/li[2]").text
    dailygawon_contents.append([news_cont_title,news_cont,news_date,dailygawon_links])
    # print(dailygawon_contents)
    
# print(dailygaewon_list)
print(dailygawon_contents)
browser.quit()
print("크롤링에 걸린시간:" time.time() - start_time) # 현재시각 -시작시간 =실행시간

```

※오류

Pyspark에서 셀레니움을 모듈을 불러오지못하는 오류를만났다.
하지만 일반 python에서는 셀레니움 모듈이 잘 불러와졌다.

마스터,워커1,2 파이썬버전, 셀레니움버전, 에어플로우 버전 모두 같으며 실행환경도 에어플로우 가상환경에서 실행했는데 왜 못찾아오는건지 이해되지않아서 불러오는 경로를 확인해봤다.

```python

일반 python

import selenium
import requests
import inspect
print(inspect.getfile(selenium))

-> 
/home/{유저명}/anaconda3/envs/env명/lib/python3.9/site-packages/selenium/__init__.py

->
/home/{유저명}/anaconda3/envs/env명/lib/python3.9/site-packages/requests/__init__.py

위와같이 경로를 반환한다. 그럼 pyspark에서 불러와지는 모듈의 경로와 셀레니움이같은곳에있으면 경로의 문제가아니라고 생각했다.

그래서 requests모듈은 pyspark,일반 python 두곳 모두에서 잘 불러와져서 이 모듈을가지고 각각 pyspark와 python경로를 비교해보았다.


pyspark

import requests
import inspect
print(inspect.getfile(selenium))

->
/usr/lib/python3/dist-packages/requests/__init__.py

경로가 다르다.
아무래도 pyspark가 참고하는 경로에 셀레니움이없어서 못불러오는것같다.
그렇가면 두가지방법을 할수있다.

1.pyspark가 참조하는 경로를바꾼다.
2.pyspark가 참조하는 경로에 셀레니움을 옮겨준다.

아나콘다위에 올려진 가상환경에서 airflow를 실행할것이고, pip, conda를 통해 다운로드받을때마다 패키지를 옮겨주긴 너무 귀찮은일이다.. 그래서 나는 pyspark가 참조하는 경로를 바꿔주기로했다.

export PYSPARK_PYTHON =/usr/bin/python3 (기존)
 -> home/{유저명}/anaconda3/envs/{가상환경명}/bin/python3.9

export PYSPARK_DRIVER_PYTHON =/usr/bin/python3 (기존)
-> home/{유저명}/anaconda3/envs/{가상환경명}/bin/python3.9

pyspark가 참조하여 실행될 환경을 지정해주는 부분인것같다. 

export PYSPARK_DRIVER_PYTHON_OPTS="jupyternotebook"
만약 주피터 노트북에서 pyspark를 사용하려면 저기에 주피터노트북을 작성해주면된다.

이제 pyspark를 켜서 셀레니움을 작동시켜보자

import selenium
>>

잘된다! 아나콘다위의 파이썬과 기존의 파이썬이자꾸섞여서 꽤 오랜시간 시간을 잡아먹었다....
이제 다른 오류를 해결한다면 데이터 파이프라인이 완성될것같다.


이제 hdfs와 spark가 잘 연결되어있는지, 상호작용을하는지 확인하기위해 hdsf에 파일을 가공하여 올리고 불러와보겠다.

여기서 "hdfs:://ip:포트"는, 하둡 마스터노드의 내부통신(데몬)포트이다. 웹UI로 접속하는 포트로하면안된다...

 data = spark.read.csv("hdfs://<ip>:<port><hdfs파일경로>/test.csv", header ="true", inferSchema="true")
 
 data.show(30)

 +-------------------+--------------------------+-------------------------------+----------------+----------------+
|        artist_name|                store_name|                        address|             lot|             lat|
+-------------------+--------------------------+-------------------------------+----------------+----------------+
| Rebornraum_Leather|                  리본라움|  서울 송파구 마천로 148 (오...|127.135710129128|37.5041950340569|
|         마렌드로잉|                마렌드로잉|서울 마포구 모래내로 5 (성산...|126.903687766741|37.5642548691608|
.
.
..

위와같이 잘불러와진다. 하지만 불편하다. 재플린 노트북으로 불러와서 사용해도되지만 vscord로 불러도될것같긴한데..조금 고민이된다.

일단 30개만 불러온것을 저장해서 hdfs에 올려보자

data.coalesce(1).write.format("com.databricks.spark.csv").option("header", "true").save("hdfs://master:9000/user/
spark_test/Idus_save_test.csv")

coalesce(1) -> 일단1로했지만 나는 분산환경이니 나중에 늘려야겠다.
일단 테스트는 1로한다.

하둡에 들어갑니 잘 저장되어있다.


이제 위의 크롤링+전처리+저장코드를 실행시켜보자.
하지만 위의 코드는 크롤링이 잘 되어가고있는지 과정을 보기가어렵다. 물론 print문으로 하나의 기사를 가져올때마다 출력하게해도되지만, 사실 재플린노트북이나 주피터노트북을 사용하여 이용하는편이 상태를 알아보기가 더 편하긴하다.

아래는 한개의 사이트에서 기사를 크롤링+전처리+저장 까지하는 코드이다.
이 코드로 미리 테스트를해보고 다른 사이트도 추가하여, 에어플로우로 각 워커에 작업을 나눠줄것이다.

#테스트 코드 전체

    #Daily Gaewon
    #반려동물 상식을 반려동물기사 사이트에서 크롤링

    import selenium
    from selenium import webdriver
    from selenium.webdriver.firefox.service import Service
    from selenium.webdriver.common.by import By
    from selenium.webdriver.common.keys import Keys
    from selenium.webdriver import FirefoxOptions
    from selenium.common.exceptions import NoSuchElementException
    from selenium.webdriver.support.ui import WebDriverWait
    from selenium.webdriver.support import expected_conditions as EC
    from pyspark.sql import *
    from pyspark.sql.functions import regexp_replace
    from pyspark.sql.functions import col
    import re
    import time
    from pyspark.context import SparkContext
    from pyspark.sql.session import SparkSession


    #spark사용을위한 세션생성. 나는 pyspark 를 master(yarn)으로 실행해볼것이다.

    spark = SparkSession.builder.master("yarn").appName("deliy_test").getOrCreate()
    # .master("local[*]") = 연결할 Spark master URL을 설정한다. ex) 로컬에서 실행될때 "local", 4개의 core로 로컬에서 실행될때 "local[4]". []가 코어갯수

    #.appName=s Spark Web UI에서 보여질 어플리케이션 이름.
    

    # 시간 측정을위한 start_time선언
    start_time=time.time()

    webdriver_service = Service('/home/napetservicecloud/geckodiver') #(윈도우)경로를 \ 로 복사하지말고, /로변경해줄것.

    opts = FirefoxOptions()
    opts.add_argument("--headless")
    # opts.binary_location = r'C:\Program Files\Mozilla Firefox\firefox.exe'#(윈도우)에러,moz:firefoxOptions.binary를 해결하기위해, 파이어폭스 경로를 수동지정.
    browser = webdriver.Firefox(options=opts)
    #
    # 한 페이지내에서 모든 뉴스의 갯수만큼 타이틀 / 링크를 따로추출하여 두개의 리스트에 저장.
    dailygaewon_list=[] #뉴스정보를 저장할 리스트
    dailygawon_link=[] #본문링크만 저장할 리스트(본문크롤링에 필요하기에 따로 저장)

    #페이지를 순환할 for문 / 데일리개원에 미디어 페이지를 크롤링해온다.
    for pages in range(1,8): #7페이지까지있으니까 7+1=8 / 또한 원래 페이지의 URL이 page={n}&total=nnn&sc_section_code=~이지만,total은 늘 변하는값이므로, 빼줬다.빼줘도 작동지장x
        base_url = f"http://www.dailygaewon.com/news/articleList.html?page={pages}&sc_section_code=&sc_sub_section_code=S2N30&sc_serial_code=&sc_area=&sc_level=&sc_article_type=&sc_view_level=&sc_sdate=&sc_edate=&sc_serial_number=&sc_word=&box_idxno=&sc_multi_code=&sc_is_image=&sc_is_movie=&sc_order_by=E"
        browser.get(base_url)

        #페이지 내의 모든 기사의 갯수를추출하여 변수로 저장
        all_news=browser.find_elements(By.CLASS_NAME,"table-row")
        all_news_num=len(all_news)
        
        #기사의 길이만큼 for문을 반복해서 타이틀, 링크를 따로 추출
        for all_news_n in range(1,all_news_num+1): #1~19까지만 작동되니까 +1을해서 갯수를맞춰줌
            news_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a/strong").text
            news_link=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a").get_attribute("href")
            dailygaewon_list.append([news_title,news_link])
            dailygawon_link.append(news_link) #이것은 데이터 프레임으로 만들지 x
    print(len(dailygaewon_list))
    print(len(news_link))


    #링크도 데이터프레임에 추가했음.
    dailygawon_contents=[]
    for dailygawon_links in dailygawon_link:
        browser.get(dailygawon_links)
        news_cont_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/div/div").text
        news_cont=browser.find_element(By.CSS_SELECTOR,"div#article-view-content-div").text
        news_date=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/section/div/ul/li[2]").text
        dailygawon_contents.append([news_cont_title,news_cont,news_date,dailygawon_links])
        print(dailygawon_contents)
        
    # print(dailygaewon_list)
    print(dailygawon_contents)
    print(len(dailygawon_contents))
    browser.quit()   

    #========크롤링종료=========

    #Spark 가공,전처리부분

    dailygaewon_index = ["title","main","date","url"]
    dailygaewon_data= spark.createDataFrame(dailygawon_contents,dailygaewon_index)
    dailygaewon_data.createOrReplaceTempView("dailygaewon")
    dailygaewon_data.show()

    #main컬럼
    main_p=dailygaewon_data.select(col("title"),regexp_replace(col("main"),'[■\n※-▶◆▼●▲『』]'," ").alias("main"),regexp_replace(col("date"),'승인',"").alias("date"),col("url"))

    #title컬럼
    title_p=main_p.select(regexp_replace(col("title"),'\[[^)]*\]',"").alias("title"),col("main"),regexp_replace(col("date"),'\[[^)]*\]',"").alias("date"),col("url"))
    title_p.show()

    #특정문자를 삭제하고 DF에 id를 부여
    dailygaewon_wordcount_x=title_p.rdd.zipWithIndex().toDF()
    final_dailygaewon_df_wordx=dailygaewon_wordcount_x.select(col("_2").alias("id"),col("_1.*")) #_2가 id. id가앞에오게 _2를 먼저부름
    final_dailygaewon_df_wordx.show()

    #하둡에저장할때 파티션을 나누지않고 병합
    final_dailygaewon_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/dailygaewon_final_url.csv")

    print("총 걸린시간은 : ",time.time()-start_time)

    #세션종료
    spark.stop()

#실행
spark-submit deliy_test.py

#결과
총 걸린시간은 :  296.2208170890808 (단위 초. 약5분)

hdfs에도 잘 저장되는것을 확인했다. 하지만 조금 찾아보다가 spakr-submit을 사용할때 여러모드가있다는것을 알게되었다. 
deploy-mode를 지정하지않으면 클라이언트 모드가 디폴트라고한다.
그래서 클러스터 모드로 실행하면 무엇이달라질지 궁금해서 클러스터 모드로 실행시켜보았다.
```

![spark-submit_Test1](./spark/spark_submit_Test.PNG)

```python
#실행2
spark-submit --master yarn --deploy-mode cluster deliy_test.py

#결과2
         client token: N/A
         diagnostics: N/A
         ApplicationMaster host: slave02
         ApplicationMaster RPC port: 38851
         queue: default
         start time: 1694188132208
         final status: SUCCEEDED
         tracking URL: http://master:8088/proxy/application_1694186077256_0002/
         user: user_name
2023-09-08 15:53:55,135 INFO util.ShutdownHookManager: Shutdown hook called
2023-09-08 15:53:55,137 INFO util.ShutdownHookManager: Deleting directory /tmp/spark-1c2f87e9-3eca-47ab-8b73-8e42350dfe25
2023-09-08 15:53:55,162 INFO util.ShutdownHookManager: Deleting directory /tmp/spark-502b6bf5-a496-4fe0-85eb-22adaf01f71e

결과화면은 전혀 다르게나왔지만, hdfs에 제대로 작성된것은 확인하였다.

또한 slave01,slave02둘 다 각각의 메모리를 사용하는것을 확인하였다.
그래서 start 시간과 finish 시간을 비교해보니 5분으로 두개다 비슷하게나온걸 알수있었다.

하지만 아직도 궁금증이 덜풀려서 spark conf에서 설정한것과 spark-submit 쉘 명령어 옵션이 뭐가 다른거지? 라고 찾아보니 spark-submit 쉘 명령어가 가장 우선순위가 높게 처리된다는것을알수있었다.

어쩐이 python내부에서 .appname을 지정하고 spark-submit 쉘 명령어에서 --name옵션을 사용해서 이름을 지정했을때 쉘 명령어에서 지정한 이름으로 웹에 제출되는것을알수있었다.

그렇지만 이로써 알게된것은 Spark 내부에서 명령어를 처리하는 우선순위는 Shell > 파일 > Conf 기본설정 순으로 지정되어있다는것을 알수있었다.

하둡에 저장된 파일을 SPARK로 읽어와보자

+---+-------------------------------------+----------------------------------+----------------+--------------------+
| id|                                title|                              main|            date|                 url|
+---+-------------------------------------+----------------------------------+----------------+--------------------+
|  0|       잇따르는 고양이 AI 확진 ‘비...|  네이처스로우 사료 2종 제품 회...|2023.08.11 15:00|http://www.dailyg...|
|  1|      '반려견 사업 고수익 보장' 폰...| 반려견 테마파크 등 반려동물 플...|2023.07.23 08:00|http://www.dailyg...|
|  2|      보호소 탈을 쓴 ‘신종 펫숍’ 논란|동물보호소를 사칭한 신종 펫숍들...|2023.07.11 08:00|http://www.dailyg...|
|  3|     “해외 신약 빠른 도입 체계개선...|      지난 5월 19일 열린 ‘제3차...|2023.06.13 08:00|http://www.dailyg...|
|  4|      캣맘이 둔 사료로 ‘저녁 한 끼...| 길고양이 사료 먹이는 견주 행동...|2023.04.21 12:11|http://www.dailyg...|
|  5|     우군(友軍) 많지 않은 수의계 단체|최근 정부가 인체의약품 제조시설...|2023.04.21 11:18|http://www.dailyg...|
|  6| 동물의료개선 위한 동물병원 희생 안돼|농림축산식품부가 ?동물의료 개선...|2023.04.12 07:00|http://www.dailyg...|
|  7|              “도대체 개는 왜 키우나”| 잇따르는 반려동물 학대 사건…수...|2023.03.28 09:00|http://www.dailyg...|
|  8|       2% 아쉬운 ‘반려산업동물의료팀’|지난해 말 농림축산식품부는 조직...|2023.03.24 11:49|http://www.dailyg...|
|  9|       ‘태종 이방원’ 말 학대 사건,...|  검역본부, 동물 학대 여부 판단...|2023.03.09 14:04|http://www.dailyg...|
| 10|     증가하는 수의대 중도 탈락, 해...|학령인구는 급격히 감소하고 있고...|2023.03.09 13:55|http://www.dailyg...|
+---+-------------------------------------+----------------------------------+----------------+--------------------+

뉴스 사이트에 들어가봤더니, 제대로 크롤링되어 잘 적재된것을 확인했다.

여러 환경에서 같은 python파일을 실행해보니, yarn모드로 분산처리되는것이 가장 이상적이라고 판단되어 spark-submit의 클러스터 모드와 함께 사용해야겠다.


# Django 웹 서비스가 이용중인 MysqlDB에 적재하는 코드를 추가하자

크롤링->전처리->Hadoop/mysql 적재 까지 하는 코드를 추가해보자.
위의 코드는 크롤링->전처리->Hadoop적재 까지였으니 Mysql에 데이터를 적재하는 코드만 추가하면된다.


    #spark = SparkSession.builder.master("yarn").getOrCreate() ▼ 아래처럼 바꿔줌
    spark = SparkSession.builder.config("spark.jars", "mysql-connector-java-8.0.33.jar").master("yarn").getOrCreate()

    user='mysql_user_name'
    password='mysql_user_paasword'
    url='jdbc:mysql://{ip}:{port}/{db_name}'
    driver='com.mysql.cj.jdbc.Driver'
    dbtable='table_name'

    저장할DF명.write.jdbc(url,dbtable,"append",properties={"driver":driver, "user":user,"password":password})

※Spark세션을 생성할때, .config()안에 내용을 작성하기 전 'mysql-connector-j-8.0.33.jar'가 
/usr/share/java/mysql-connector-j-8.0.31.jar 와 같이 해당 경로에 파일이있어야하며, spark-defaults.conf에 'spark.jars /usr/share/java/mysql-connector-j-8.0.31.jar' 해당 내용을 추가해놓으면 좋다.(이건 선택.)

단, spark-defaults.conf에 mysql-connector에 대한 내용을 추가하지않았으면 spark-submit을 사용할때 별도로 connector를 지정해줘야한다.

#spark-submit이용
    --master yarn 
    --jars /path/to/mysql/connector/mysql-connector-java-8.0.33.jar
    OR --packages mysql:mysql-connector-java:8.0.33 로 이용할수있다.

# 번외로 mysql->spark 로 불러올때, 아래와같이 .option("query"~)를 이용해 sql쿼리문을 사용하여 원하는데이터만 가져올수있다.

    df = spark.read \
    .format("jdbc") \
    .option("query", "select id,age from employee where gender='M'")
```

### 완성된 테스트 코드

아래 코드는 전체 코드중 한개의 뉴스사만 크롤링하고 전처리, 적재하는 코드이므로 이제 이 코드를 토대로 여러 뉴스사의 구조에맞게 코드를 작성하여 airflow를통해 총 4개의 뉴스사에서 뉴스를 수집해올것이다.

```python
#Daily Gaewon
#반려동물 상식을 반려동물기사 사이트에서 크롤링

import selenium
from selenium import webdriver
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import FirefoxOptions
from selenium.common.exceptions import NoSuchElementException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from pyspark.sql import *
from pyspark.sql.functions import regexp_replace
from pyspark.sql.functions import col
import re
import time
from pyspark.context import SparkContext
from pyspark.sql.session import SparkSession



#spark세션생성

spark = SparkSession.builder.config("spark.jars", "mysql-connector-java-8.0.33.jar").master("yarn").getOrCreate()

#시간측정
start_time=time.time()

#webdriver_service = Service('/home/napetservicecloud/geckodiver') #(윈도우)경로를 \ 로 복사하지말고, /로변경해줄것.

opts = FirefoxOptions()
opts.add_argument("--headless")
# opts.binary_location = r'C:\Program Files\Mozilla Firefox\firefox.exe'#(윈도우)에러,moz:firefoxOptions.binary를 해결하기위해, 파이어폭스 경로를 수동지정.
browser = webdriver.Firefox(options=opts)
#
# 한 페이지내에서 모든 뉴스의 갯수만큼 타이틀 / 링크를 따로추출하여 두개의 리스트에 저장.
dailygaewon_list=[] #뉴스정보를 저장할 리스트
dailygawon_link=[] #본문링크만 저장할 리스트(본문크롤링에 필요하기에 따로 저장)

#페이지를 순환할 for문 / 데일리개원에 미디어 페이지를 크롤링해온다.
for pages in range(1,8): #7페이지까지있으니까 7+1=8 / 또한 원래 페이지의 URL이 page={n}&total=nnn&sc_section_code=~이지만,total은 늘 변하는값이므로, 빼줬다.빼줘도 작동지장x
    base_url = f"http://www.dailygaewon.com/news/articleList.html?page={pages}&sc_section_code=&sc_sub_section_code=S2N30&sc_serial_code=&sc_area=&sc_level=&sc_article_type=&sc_view_level=&sc_sdate=&sc_edate=&sc_serial_number=&sc_word=&box_idxno=&sc_multi_code=&sc_is_image=&sc_is_movie=&sc_order_by=E"
    browser.get(base_url)

    #페이지 내의 모든 기사의 갯수를추출하여 변수로 저장
    all_news=browser.find_elements(By.CLASS_NAME,"table-row")
    all_news_num=len(all_news)
    
    #기사의 길이만큼 for문을 반복해서 타이틀, 링크를 따로 추출
    for all_news_n in range(1,all_news_num+1): #1~19까지만 작동되니까 +1을해서 갯수를맞춰줌
        news_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a/strong").text
        news_link=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a").get_attribute("href")
        dailygaewon_list.append([news_title,news_link])
        dailygawon_link.append(news_link) #이것은 데이터 프레임으로 만들지 x
print(len(dailygaewon_list))
print(len(news_link))


#링크도 데이터프레임에 추가했음.
dailygawon_contents=[]
for dailygawon_links in dailygawon_link:
    browser.get(dailygawon_links)
    news_cont_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/div/div").text
    news_cont=browser.find_element(By.CSS_SELECTOR,"div#article-view-content-div").text
    news_date=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/section/div/ul/li[2]").text
    news_press="DailyGaewon" #뉴스사 추가
    dailygawon_contents.append([news_cont_title,news_cont,news_press,dailygawon_links,news_date])
print(dailygawon_contents)
    
# print(dailygaewon_list)
print(dailygawon_contents)
print(len(dailygawon_contents))
browser.quit()   

#========크롤링종료=========

#Spark 가공,전처리부분

dailygaewon_index = ["title","main","press","url","write_date"]
dailygaewon_data= spark.createDataFrame(dailygawon_contents,dailygaewon_index)
dailygaewon_data.createOrReplaceTempView("dailygaewon")
dailygaewon_data.show()

#main컬럼
main_p=dailygaewon_data.select(col("title"),regexp_replace(col("main"),'[■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'승인',"").alias("write_date"))

#title컬럼
title_p=main_p.select(regexp_replace(col("title"),'\[[^)]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
#title_p.show()

#특정문자를 삭제하고 DF에 id를 부여
dailygaewon_wordcount_x=title_p.rdd.zipWithIndex().toDF()
final_dailygaewon_df_wordx=dailygaewon_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*")) #_2가 id. id가앞에오게 _2를 먼저부름. 그리고 0부터 부여되는 id값 전체에 1을 더해줘서 1부터 시작하게만든다.

#final_dailygaewon_df_wordx.show()

#하둡에저장할때 파티션을 나누지않고 병합
final_dailygaewon_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/dailygaewon_final_url.csv")

#Mysql에 저장
user='user_name'
password='user_password'
url='jdbc:mysql://<ip>:<port>/<db_name>'
driver='com.mysql.cj.jdbc.Driver'
dbtable='table_name'

final_dailygaewon_df_wordx.write.jdbc(url,dbtable,"append",properties={"driver":driver, "user":user,"password":password})

#appen = 추가
#overwrite = 덮어쓰기
#errorifexists = 이미 다른파일이 존재할경우 오류를 발생시킴

#save()는 스칼라용. pyton을사용하는 나에게는 x

# 총 걸린시간 출력문
print("총 걸린시간은 : ",time.time()-start_time)

#스파크 세션종료
spark.stop()
```
![spark-submit-mysql](./spark/spark_submit_mysql_save1.PNG)

mysql에 수집한 데이터가 spark전처리 과정을 거처 잘 적재되었다.

그렇다면, 이제 언론사를 더 추가하여 여러곳에서 데이터를 수집해오자.

초기엔 언론사1+_언론사2+언론사3+...=기사테이블  의 구조로 언론사별로 따로 저장하고 후에 취합하는 구조로 설계하였는데

전처리과정이 비슷한 부분이많아서 한개의 DF에 언론사1,2,3의 수집데이터를 한번에 추가하고 하나의 기사테이블로 이용할것이다.
만약 언론사별로 가공이 필요하다면 press로 기사를 언론사별로 구분할수있기때문에 가공이 필요할때 분리해서 가공하는 방법을 채택했다.

하지만 꽤 테스트하는데 시간이 걸려서, 시간을 단축하고자 Spark가 사용할수있는 코어 및 메모리를 조절해줬다.
Standalon 과 yarn 모드 둘다 설정해줬다.

<u>spark 메모리 및 코어 설정 </u>

```bash

#[master,worker노드 전부]
#spark-defaults.conf

spark.master            yarn
spark.eventLog.enabled  true
spark.eventLog.dir      hdfs://master:9000/spark
spark.jars              /usr/share/java/mysql-connector-j-8.0.33.jar
spark.driver.memory     6g


#[master노드만]

# web master

export SPARK_MASTER_HOST=master
export SPARK_MASTER_WEBUI_PORT=18080

# worekr-Standalone

export SPARK_WORKER_DIR=${SPARK_WORKER_DIR:-$SPARK_HOME/work}
export SPARK_WORKER_WEBUI_PORT=18081
export SPARK_WORKER_INSTANCES=2
export SPARK_WORKER_CORES=4
export SPARK_WORKER_MEMORY=6g


# yarn mode Memory

export SPARK_DRIVER_MEMOR=6g
export SPARK_EXECUTOR_MEMORY=6g
export SPARK_EXECUTOR_CORES=4

#python ->이 옵션같은경우에는 마스터에만 해주면된다고하지만,나는 워커에서도 잠깐 pyspark를 쓸일있을까싶어서설정해둠

export PYSPARK_PYTHON={path}/anaconda3/envs/{env_name}/bin/python3.9
export PYSPARK_DRIVER_PYTHON={path}/anaconda3/envs/{env_name}/bin/python3.9

```

설정 전 기본값으로 test code를 실행해본결과 = 33분

설정 후 test code를 실행해본결과 = 12분

굉장히 빨라진것을 확인할수있었다.

하지만 8GB이상으로 메모리를 할당하였더니, 에러를 반환한다.
보니까 실행구조가 YARN모드를 통해 SPARK-SUBMIT을 실행시켜 신청서를 제출하면 설정값에따라 YARN에서 SPARK_APP을 실행시킬수있는 컨테이너를 제공한다. 또한 이 컨테이너의 기본값은 **8GB**이다. 

기본값을 바꿔주려면 <u>yarn-site.xml</u>을 편집해줘야한다. 

그러므로 spark의 메모리와 코어수를 지정할때는 한개의 노드가 사용할수있는 최대 메모리로 설정하면안되고 어느정도 여유를 두고 설정해줘야한다.(하둡이나 os등 spark외의 프로그램들도 메모리를 사용해야하니까)

    아래와같이 spark-env.sh를 설정한다면 익스큐터는 한개, 하나의 익스큐더가 사용할수있는 메모리는 18, 드라이버가 사용하는 메모리는 4 가되므로 해당 노드의 한개의 익스큐터가 총 18+4 =22의 메모리를 사용하게된다.

    SPARK_EXECUTOR_INSTANCES = 1
    SPARK_EXECUTOR_MEMORY = 18G
    SPARK_DRIVER_MEMORY = 4G

또한, yarn에서 컨테이너를 제공할때 0.5gb정도의 여유를 두고 제공하는것같다.
위처럼 18g를 사용한다고 설정해두면 18.5정도가되는것

yarn-site.xml -> yarn.scheduler.maximum-allocation-mb 옵션은 **하나의 컨테이너의 메모리**를 설정해주는 옵션이다.

yarn.nodemanager.resource.memory-mb 옵션은 **전체 컨테이너의 메모리**를 설정해주는 옵션이다.

즉 노드매니저 리소스메모리가 10G이면 1G짜리 컨테이너를 10개만들수있고, 2G짜리 컨테이너를 5개만들어 제공할수있다.
스케쥴러 맥시멈 옵셥은 하나의 컨테이너가 가질수있는 최대값이다. 기본값 8G이니까 8GB이상을 요구하는 앱이나 설정을 실행시킬수없다.

나는 아래와같이 설정해주었다.

    [yarn-site.xml]
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>12000</value>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>10000</value>
    </property>


실험과 실행을하다보니 장기간 웹작업에 pyspark를 이용한 spark-submit이 비효율적이라는 사실을 알게되었다.
그래서 장시간 웹작업인 <u>크롤링은 airflow를 통해 워커들에게시키고, 수집된 기사를 spark-submit을통해 전처리만 하기로하였다.</u>


그러므로 나는 아래와같이 코드를 작성하였다.

```python

#반려동물 상식을 반려동물기사 사이트에서 크롤링

import selenium
from selenium import webdriver
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import FirefoxOptions
from selenium.common.exceptions import NoSuchElementException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from pyspark.sql import *
from pyspark.sql.functions import regexp_replace
from pyspark.sql.functions import col
import re
import time
from pyspark.context import SparkContext
from pyspark.sql.session import SparkSession
import pandas as pd


#=============SPARK세션생성=============

spark = SparkSession.builder.config("spark.jars", "mysql-connector-java-8.0.33.jar").master("yarn").getOrCreate()

#=============Daily Gaewon 크롤링 =============

#시간측정
start_time=time.time()

#webdriver_service = Service('/home/napetservicecloud/geckodiver') #(윈도우)경로를 \ 로 복사하지말고, /로변경해줄것.

opts = FirefoxOptions()
opts.add_argument("--headless")
# opts.binary_location = r'C:\Program Files\Mozilla Firefox\firefox.exe'#(윈도우)에러,moz:firefoxOptions.binary를 해결하기위해, 파이어폭스 경로를 수동지정.
browser = webdriver.Firefox(options=opts)
#
# 한 페이지내에서 모든 뉴스의 갯수만큼 타이틀 / 링크를 따로추출하여 두개의 리스트에 저장.
dailygaewon_list=[] #뉴스정보를 저장할 리스트
dailygawon_link=[] #본문링크만 저장할 리스트(본문크롤링에 필요하기에 따로 저장)

#페이지를 순환할 for문 / 데일리개원에 미디어 페이지를 크롤링해온다.
for pages in range(1,8): #7페이지까지있으니까 7+1=8 / 또한 원래 페이지의 URL이 page={n}&total=nnn&sc_section_code=~이지만,total은 늘 변하는값이므로, 빼줬다.빼줘도 작동지장x
    base_url = f"http://www.dailygaewon.com/news/articleList.html?page={pages}&sc_section_code=&sc_sub_section_code=S2N30&sc_serial_code=&sc_area=&sc_level=&sc_article_type=&sc_view_level=&sc_sdate=&sc_edate=&sc_serial_number=&sc_word=&box_idxno=&sc_multi_code=&sc_is_image=&sc_is_movie=&sc_order_by=E"
    browser.get(base_url)

    #페이지 내의 모든 기사의 갯수를추출하여 변수로 저장
    all_news=browser.find_elements(By.CLASS_NAME,"table-row")
    all_news_num=len(all_news)
    
    #기사의 길이만큼 for문을 반복해서 타이틀, 링크를 따로 추출
    for all_news_n in range(1,all_news_num+1): #1~19까지만 작동되니까 +1을해서 갯수를맞춰줌
        news_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a/strong").text
        news_link=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a").get_attribute("href")
        dailygaewon_list.append([news_title,news_link])
        dailygawon_link.append(news_link) #이것은 데이터 프레임으로 만들지 x
#print(len(dailygaewon_list))
#print(len(news_link))


#타이틀/ 본문 / 언론사/ url / 작성날짜

dailygawon_contents=[]
for dailygawon_links in dailygawon_link:
    browser.get(dailygawon_links)
    news_cont_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/div/div").text
    news_cont=browser.find_element(By.CSS_SELECTOR,"div#article-view-content-div").text
    news_date=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/section/div/ul/li[2]").text
    news_press="DailyGaewon" #뉴스사 추가
    dailygawon_contents.append([news_cont_title,news_cont,news_press,dailygawon_links,news_date])
#print(dailygawon_contents)
    
print(dailygaewon_list)
print(dailygawon_contents)
#print(len(dailygawon_contents))
browser.quit()   

#=============Kukmin Ilbo 크롤링 =============
	
browser_2 = webdriver.Firefox(options=opts)

# #=============네이버 페이지 크롤링용 변수선언(국민,KBS,SBS공통)=============

pagenum=[1,11,21,31,41,51,61,71,81,91,101]

#=============1-2. 국민일보 크롤링=============

#링크 크롤링

Kukmin_link=[] #국민일보 본문링크만 담을 리스트

for links in pagenum:
    Kukmin_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=20&mynews=1&office_type=1&office_section_code=1&news_office_checked=1005&nso=so:r,p:all,a:all&start={links}"
    browser_2.get(Kukmin_pagelink)
    all_page_lang=browser_2.find_elements(By.CLASS_NAME,"dsc_thumb")
    for cont in all_page_lang:
        Kukmin_content_link=cont.get_attribute("href") #본문페이지 링크 추출
        Kukmin_link.append(Kukmin_content_link)
        #print(Kukmin_link)
#print(len(Kukmin_link))


# #타이틀/ 본문 / 언론사/ url / 작성날짜

Kukmin_main_text=[]

for Kukmin_links in Kukmin_link:
    browser_2.get(Kukmin_links) #본문페이지로 변경
    try:
        Kukmin_title=browser_2.find_element(By.CSS_SELECTOR, "div.nwsti h3").text
        time.sleep(1)#로딩으로인해 발생하는 error를 줄이기위해 1초 대기
        Kukmin_main=browser_2.find_element(By.CLASS_NAME, "tx").text
        Kukmin_date=browser_2.find_element(By.CLASS_NAME, "t11").text
        Kukmin_news_press="KukminIlbo"
        Kukmin_main_text.append([Kukmin_title,Kukmin_main,Kukmin_news_press,Kukmin_links,Kukmin_date])
        #print(Kukmin_main_text)
    except:
        pass #daily gaewon은 에러가나지않았지만, kbs,sbs,kukmin은 오류가있음
#print(Kukmin_main_text)
#print(len(Kukmin_main_text))

browser_2.quit()

#============= KBS =============

browser_3 = webdriver.Firefox(options=opts)

# 링크

kbs_link=[] # KBS 본문링크만 담을 리스트

for links in pagenum:
    kbs_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=28&mynews=1&office_type=1&office_section_code=2&news_office_checked=1056&nso=so:r,p:all,a:all&start={links}"
    browser_3.get(kbs_pagelink)
    all_page_lang=browser_3.find_elements(By.CLASS_NAME,"dsc_thumb")
    for cont in all_page_lang:
        kbs_content_link=cont.get_attribute("href")
        kbs_link.append(kbs_content_link)

# 본문내용

#타이틀/ 본문 / 언론사/ URL / 작성날짜

Kbs_main_text=[]

for kbs_links in kbs_link:
    try:
        browser_3.get(kbs_links) #본문페이지로 변경
        time.sleep(1)#로딩으로인해 발생하는 error를 줄이기위해 1초 대기
        kbs_title=browser_3.find_element(By.CLASS_NAME, "tit-s").text
        kbs_main=browser_3.find_element(By.ID, "cont_newstext").text
        kbs_date=browser_3.find_element(By.CLASS_NAME, "date").text
        kbs_news_press="KBS"
        Kbs_main_text.append([kbs_title,kbs_main,kbs_news_press,kbs_links,kbs_date])
    except:
        pass
print(Kbs_main_text)

browser_3.quit()

#============= SBS =============

#네이버 페이지 크롤링용 변수

browser_4 = webdriver.Firefox(options=opts)


sbs_link=[] 
for links in pagenum:
    sbs_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=22&mynews=1&office_type=1&office_section_code=2&news_office_checked=1055&nso=so:r,p:all,a:all&start={links}"
    browser_4.get(sbs_pagelink)
    all_page_lang=browser_4.find_elements(By.CLASS_NAME,"dsc_thumb")
    for cont in all_page_lang:
        sbs_content_link=cont.get_attribute("href") 
        sbs_link.append(sbs_content_link)

# #======1-2. SBS 크롤링 / 본문내용=============

# #타이틀/ 본문 / 입력날짜 /URL

sbs_main_text=[]

for sbs_links in sbs_link:
    try:
        browser_4.get(sbs_links) 
        time.sleep(1)
        sbs_title=browser_4.find_element(By.ID, "news-title").text
        sbs_main=browser_4.find_element(By.CLASS_NAME, "text_area").text
        sbs_date=browser_4.find_element(By.CSS_SELECTOR, "div.date_area span").text
        sbs_press="SBS"
        sbs_main_text.append([sbs_title,sbs_main,sbs_press,sbs_links,sbs_date])
    except:
        pass
#print(sbs_main_text)

browser_4.quit()

#================크롤링종료================

#================Daily Gaewon Spark 가공,전처리부분================

dailygaewon_index = ["title","main","press","url","write_date"]
dailygaewon_data= spark.createDataFrame(dailygawon_contents,dailygaewon_index)
dailygaewon_data.createOrReplaceTempView("dailygaewon")
#dailygaewon_data.show()

#main컬럼
diy_main_p=dailygaewon_data.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'승인',"").alias("write_date"))

#title컬럼
diy_title_p=diy_main_p.select(regexp_replace(col("title"),'\[[^)]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
#diy_title_p.show()

#특정문자를 삭제하고 DF에 id를 부여
dailygaewon_wordcount_x=diy_title_p.rdd.zipWithIndex().toDF()
final_dailygaewon_df_wordx=dailygaewon_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*")) #_2가 id. id가앞에오게 _2를 먼저부름
final_dailygaewon_df_wordx.show()

#dailygaewon//하둡에저장할때 파티션을 나누지않고 병합
final_dailygaewon_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/dailygaewon_final_url.csv")

#================Kukmin Ilbo 가공,전처리부분================
Kukmin_index = ["title","main","press","url","write_date"]
Kukmin_news = spark.createDataFrame(Kukmin_main_text, Kukmin_index)
Kukmin_news.createOrReplaceTempView("Kukmin")
#Kukmin_news.show()

#main컬럼 전처리 ex)[\n▶▲■▷▼●]'," "
kuk_main_p=Kukmin_news.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'승인',"").alias("write_date"))

# title 컬럼
kuk_title_p=kuk_main_p.select(regexp_replace(col("title"),'\[[^),]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
#kuk_title_p.show()

# #id생성문
kukmin_wordcount_x=kuk_title_p.rdd.zipWithIndex().toDF()
final_kukmin_df_wordx=kukmin_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*"))
final_kukmin_df_wordx.show()

#하둡저장
final_kukmin_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/Kukmin_news_url.csv")

#================KBS 가공,전처리부분================

Kbs_index = ["title","main","press","url","write_date"]
Kbs_news = spark.createDataFrame(Kbs_main_text, Kbs_index)
Kbs_news.createOrReplaceTempView("Kbs_news")
#Kbs_news.show()

#main컬럼 전처리
kb_main_p=Kbs_news.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'입력',"").alias("write_date"))

#title컬럼 전처리
kb_title_p=kb_main_p.select(regexp_replace(col("title"),'\[[^),]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\([^)]*\)',"").alias("write_date"))
#kb_title_p.show()

#ID생성,CSV파일생성
kbs_wordcount_x=kb_title_p.rdd.zipWithIndex().toDF()
final_kbs_df_wordx=kbs_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*"))
final_kbs_df_wordx.show()

final_kbs_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/Kbs_news_url.csv/")


#================SBS 가공,전처리부분================

sbs_index = ["title","main","press","url","write_date"]
sbs_news = spark.createDataFrame(sbs_main_text,sbs_index)
sbs_news.createOrReplaceTempView("sbs_news")
sbs_news.show()

#main컬럼 전처리
sb_main_p=sbs_news.select(col("title"),regexp_replace(col("main"),'[\n▶▲■▷▼●\[\]『』©※]'," ").alias("main"),col("press"),col("url"),col("write_date"))

# title 컬럼에대하여 정규식을 사용해 여러문자 제거. []는 특수한문자이므로 이스케이프가 필요하여 앞에 \를 사용. []안에있는 문자들과 일치하면 모두 공백으로변경
sb_title_p=sb_main_p.select(regexp_replace(col("title"),'[\[,"\]+]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
sb_title_p.show()

#id생성,csv파일 생성

sbs_rdd_id=sb_title_p.rdd.zipWithIndex().toDF()
final_sbs_wordx=sbs_rdd_id.select((col("_2")+1).alias("id"),col("_1.*"))
final_sbs_wordx.show()
final_sbs_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/sbs_news_url.csv")


#================Mysql저장을 위한 전처리 + Mysql저장================

## 데이터프레임 병합

### HDSF에 업로드된것을 불러오기
kbs_idx=final_kbs_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
sbs_idx=final_sbs_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
kukmin_idx=final_kukmin_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
daily_idx=final_dailygaewon_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))

####concat을위해 sparkDF->pandasDF->sparkDF으로 변환 작업

result_kbs = kbs_idx.select("*").toPandas()
result_sbs = sbs_idx.select("*").toPandas()
result_kukmin = kukmin_idx.select("*").toPandas()
result_daily = daily_idx.select("*").toPandas()

concatDF=pd.concat([result_daily,result_kukmin,result_sbs,result_kbs])
news_concatdf = spark.createDataFrame(concatDF)

###ID재부여

news_concat_id=news_concatdf.rdd.zipWithIndex().toDF()
final_newsDF=news_concat_id.select((col("_2")+1).alias("id"),col("_1.*"))

### 병합된파일 hdfs에 저장
final_newsDF.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/news_final.csv")

## Mysql드라이버

user='user_name'
password='passwd'
url='jdbc:mysql://<ip>:<port>/db_name'
driver='com.mysql.cj.jdbc.Driver'
dbtable='table_name'

## Mysql에 저장

final_newsDF.write.jdbc(url,dbtable,"append",properties={"driver":driver, "user":user,"password":password})

# 총 걸린시간 출력문
print("총 걸린시간은 : ",time.time()-start_time)

#스파크 세션종료
spark.stop()

```

### Airflow에서 SSH를 써보자

이 파트를 만든이유는, Airflow에서 ssh통신을 사용하려면 통신을 등록해줘야한다는것을 알아서 만들었다.

나는, airflow를 통해 worker-1,2에서 수집한 크롤링 데이터파일을 Master노드에 보내, Spark를 통해 전처리를하도록 설계하였다.

하지만, Bash Operator를 통해 worker-1 에서 **scp명령어를 사용했을때, 에러가 발생하였다.**
난 이미 마스터-워커1-워커2 사이에 포트포워딩이 끝나있고 ssh키 교환을 마쳤기때문에 당연히 될줄알았는데 그게아니였다.


그래서 찾아보니 airflow에 ssh통신을 위한 기본적인 정보를 등록해줘야한다는것을 알았다.

ssh통신을 하는 방법으로 여러 오퍼레이터가있었는데

- SSHOperator : ssh통신을 하기위해 사용되는 오퍼레이터
- SFTPOperator : sftp 통신을 하기위해 사용되는 오퍼레이터. 파일을 내보내고 받아오는데 사용됨
- SSHHook : ssh_conn_id를 대체함. 

약간 찾아본결과 각 오퍼레이터가 위와같은 상황에 사용되는것을 알수있었다.
그렇지만 내가 정말 헷갈렸던부분은 ssh통신을 위해 사용되는 오퍼레이터가아니라, 오퍼레이터 내부에서 사용되는 ssh_conn_id /ssh_hook부분이였다. 즉 ssh 통신을 위한 정보를 등록하는 부분이 조금 어려웠다.

왜냐하면 SSH KEY의 정보를 제대로 등록해도 DSA로 분류되어 너무길다는 오류를 반환하였기때문이다.

왜 이러한 오류가 생기는지 아직 찾지못했지만, 일단 SSHOOK를 사용해 오류를 해결하고 통신에 성공하였다.

ssh_conn_id 는 통신할 정보를 담고있는 id이다.

Airflow WEB >  Connection 탭에서 추가해줘야한다.

### SSH_CONN_ID을 등록하는법


![SSH_CONN_ID등록법](/airflow/SSH_CONN_ID.PNG)

각 항목을 살펴본다면

- Connection Id : SSH_COON_ID의 이름이 될 부분
- Connection Type : SSHOperator를 사용할거면 SSH, SFTP를 사용할거면 SFTP. (근데 SFTP를 사용하는데 SSH로 해도 통신은되었다.)

- HOST : ★ 연결할 HOST의 IP. 즉 Remote IP라고 생각하면된다. 로컬경로,IP는 해당 TASK가 실행되는 서버의 정보를 따르므로, 여기엔 원격지의 정보를 넣는다.
 
- USERNAME : 통신할 HOST의 유저 ID. root,계정명
- Password : Username의 passwd. 없으면 입력하지않아도 무관
- Port : SSH가 실행되고있는 포트

- Extra : SSH 통신을 사용할때 적용할 각종 옵션들을 { 키: 밸류 키2:밸류2} 형식으로 넣을수있다. 옵션은 AIRFLOW 공식홈페이지 참조. 이 부분엔 위에서 작성한 원격지에 접근할 ssh 개인키의 정보를 넣어준다.

**★EX-워커1에서 마스터노드1에 접근하려면, HOST에 마스터노드1의 IP. Extra에 워커1의 ssh개인 키 경로 및 정보를 넣어준다. (물론 마스터노드와 키 교환은 미리 되어있어야한다.)**


    나는 각 워커노드에 ssh키의 이름이 기본값도아니고 다 다르기때문에 master노드에 대한 연결을 따로 두개 생성하였다. ( master-1 , master-2 )

    [master-1]
    
    {
    "key_file":"path/.ssh/워커1의개인키명",
    "no_host_key_check": "True"
    }

    [master-2]

    {
    "key_file":"path/.ssh/워커2의개인키명",
    "no_host_key_check": "True"
    }

    원래는 SSH KEY의 경로, HOST_KEY라고해서 ./ssh 아래의 known_hosts에 등록되어있는 해당 키의 정보도 입력해줘야하지만, 나는 오히려 그것들을 입력하면 키가 DSA로 분류되어 너무 길다는 오류만을 잔뜩받았다.

    "disabled_algorithms": {"pubkeys": ["rsa-sha2-256", "rsa-sha2-512"]}, 라는 옵션을 추가하여 해당 키를 매핑해주면 문제가 해결될것같은데 나중에 실험해봐야겠다.(Known_hosts의 앞에 rsa-sha2-256, 512 같은 종류가 적혀있던데, 이것과 관련있지않을까싶다.)
    
    하지만 paramiko.ssh_exception.SSHException: No hostkey for host 오류가 난다면 

    {
    "key_file": "/usr/local/airflow/.ssh/id_rsa", 
    "no_host_key_check": true
    }

    위 처럼 키의 경로를 등록해줘야할수도있다.

[SFTP참고자료](http://www.kwangsiklee.com/2022/01/airflow-localsftp-sensor-%EC%8B%A4%EC%8A%B5%ED%95%B4%EB%B3%B4%EA%B8%B0/)

[AIRFLOW_SSH](https://airflow.apache.org/docs/apache-airflow-providers-ssh/stable/connections/ssh.html)

### SSHHOOK를 이용하는법

SSHHOOK를 이용하는 방법은 스크립트파일내에서 선언하는방법으로 AIRFLOW_WEB Connection에 ID를 등록하지않는 방법이다.

```python

from airflow import DAG
from datetime import datetime, timedelta
from airflow.providers.ssh.hooks.ssh import SSHHook

default_args = { 
    'owner' : 'name', 
    'depends_on_past': False,
    'retires': 1,
    'retry_delay': timedelta(minutes=5)
}

ssh_hook = SSHHook(remote_host='ip', username='user_name', port=ssh_port)

#remote_host -> 원격지의 ip
#username -> 연결할 유저명(root,계정명 등)
#port -> ssh 서비스가 이용중인 포트

with DAG( 
    'scp_test', #dag id
    default_args = default_args,
    schedule = '@daily',
    start_date = datetime(2023, 9, 15),#스케쥴에 올라가는시간. 시작시간X
    catchup = False,
    tags = ['sftp','ssh','test']

)as dag:

    t1 =  SSHOperator(
        task_id ="scp-test-bash",
        ssh_hook=ssh_hook,
        command="echo success!", #원격지에서 작업할 커맨드. 배쉬커맨드와 비슷한 역할을한다.
        queue = "worker-1"
    )

★만약 ssh_hook말고 ssh_conn_id를 사용할거면

    ssh_conn_id = "airflow-web에등록한 conn id명"
    
이렇게 입력해주면된다.

```

위와 같은 task를 실행하게된다면 worker-1이라는 큐 이름을 전담하는 worker들이 해당 큐에 들어온 t1의 작업을하게된다.

그렇게된다면 로컬은 worker-1이되는거고 원격지는 ssh_hook에 등록된 master가 되게된다. 그럼 master node에서 echo success! 커맨드를 입력하게되어, success!라는 출력결과가나온다.

![ssh_oprator](./airflow/ssh_oprator.PNG)


이제 news기사를 자동수집하는 데이터파이프라인의 전체적인 코드와 그림을 본다.

```python

#/*****셀레니움 관련******/

import selenium
from selenium import webdriver
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver import FirefoxOptions
from selenium.common.exceptions import NoSuchElementException
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import re
import time
import pandas as pd


#/*****에어플로우 관련*****/

from airflow import DAG
from datetime import datetime, timedelta
from airflow.contrib.operators.ssh_operator import SSHOperator
from airflow.providers.ssh.hooks.ssh import SSHHook
from airflow.operators.bash_operator import BashOperator
from airflow.operators.python_operator import PythonOperator
from email.policy import default
from textwrap import dedent
import csv
from airflow.operators.dummy import DummyOperator
from airflow.utils.task_group import TaskGroup


#=============================================================

#airflow 기본 args 선언

default_args = { 
    'owner' : 'gcp_master', 
    'depends_on_past': False,
    'retires': 1
    #'retry_delay': timedelta(minutes=5)
}

#경로,현재시간 포맷지정
path='/home/napetservicecloud'
time_stamp_format=datetime.now().strftime('%y%m%d_%H')
#=============Daily Gaewon 크롤링 =============

#시간측정
start_time=time.time()

#셀레니움 옵션지정
def dailygaewonc():
    opts = FirefoxOptions()
    opts.add_argument("--headless")
    browser = webdriver.Firefox(options=opts)

    # 한 페이지내에서 모든 뉴스의 갯수만큼 타이틀 / 링크를 따로추출하여 두개의 리스트에 저장.
    dailygaewon_list=[] #뉴스정보를 저장할 리스트
    dailygawon_link=[] #본문링크만 저장할 리스트(본문크롤링에 필요하기에 따로 저장)

    #페이지를 순환할 for문 / 데일리개원에 미디어 페이지를 크롤링해온다.
    for pages in range(1,8): #7페이지까지있으니까 7+1=8 / 또한 원래 페이지의 URL이 page={n}&total=nnn&sc_section_code=~이지만,total은 늘 변하는값이므로, 빼줬다.빼줘도 작동지장x
        base_url = f"http://www.dailygaewon.com/news/articleList.html?page={pages}&sc_section_code=&sc_sub_section_code=S2N30&sc_serial_code=&sc_area=&sc_level=&sc_article_type=&sc_view_level=&sc_sdate=&sc_edate=&sc_serial_number=&sc_word=&box_idxno=&sc_multi_code=&sc_is_image=&sc_is_movie=&sc_order_by=E"
        browser.get(base_url)

        #페이지 내의 모든 기사의 갯수를추출하여 변수로 저장
        all_news=browser.find_elements(By.CLASS_NAME,"table-row")
        all_news_num=len(all_news)
        
        #기사의 길이만큼 for문을 반복해서 타이틀, 링크를 따로 추출
        for all_news_n in range(1,all_news_num+1): #1~19까지만 작동되니까 +1을해서 갯수를맞춰줌
            news_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a/strong").text
            news_link=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/div[2]/section/article/div[2]/section/div[{all_news_n}]/div[1]/a").get_attribute("href")
            dailygaewon_list.append([news_title,news_link])
            dailygawon_link.append(news_link) #이것은 데이터 프레임으로 만들지 x
    #print(len(dailygaewon_list))
    print(len(news_link))


    #타이틀/ 본문 / 언론사/ url / 작성날짜

    dailygawon_contents=[]
    for dailygawon_links in dailygawon_link:
        browser.get(dailygawon_links)
        news_cont_title=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/div/div").text
        news_cont=browser.find_element(By.CSS_SELECTOR,"div#article-view-content-div").text
        news_date=browser.find_element(By.XPATH,f"//*[@id='user-container']/div[3]/header/section/div/ul/li[2]").text
        news_press="DailyGaewon" #뉴스사 추가
        dailygawon_contents.append([news_cont_title,news_cont,news_press,dailygawon_links,news_date])
    print(dailygawon_contents)
        
    # print(dailygaewon_list)
    # print(dailygawon_contents)
    # print(len(dailygawon_contents))

    dailygaewon_index = ["title","main","press","url","write_date"]
    daliygaewon_csv=pd.DataFrame(dailygawon_contents,columns=dailygaewon_index)
    daliygaewon_csv.to_csv(f'{path}/crawling/dailygawon_csv_{time_stamp_format}.csv', index=False,encoding="utf-8")
   
    browser.quit()


#=============Kukmin Ilbo 크롤링 =============
	
def kukminc():
    pagenum=[1,11,21,31,41,51,61,71,81,91,101]
    opts = FirefoxOptions()
    opts.add_argument("--headless")
    browser = webdriver.Firefox(options=opts)

#링크 크롤링

    Kukmin_link=[] #국민일보 본문링크만 담을 리스트

    for links in pagenum:
        Kukmin_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=20&mynews=1&office_type=1&office_section_code=1&news_office_checked=1005&nso=so:r,p:all,a:all&start={links}"
        browser.get(Kukmin_pagelink)
        all_page_lang=browser.find_elements(By.CLASS_NAME,"dsc_thumb")
        for cont in all_page_lang:
            Kukmin_content_link=cont.get_attribute("href") #본문페이지 링크 추출
            Kukmin_link.append(Kukmin_content_link)
    #print(len(Kukmin_link))


    #타이틀/ 본문 / 언론사/ url / 작성날짜

    Kukmin_main_text=[]

    for Kukmin_links in Kukmin_link:
        browser.get(Kukmin_links) #본문페이지로 변경
        try:
            Kukmin_title=browser.find_element(By.CSS_SELECTOR, "div.nwsti h3").text
            time.sleep(1)#로딩으로인해 발생하는 error를 줄이기위해 1초 대기
            Kukmin_main=browser.find_element(By.CLASS_NAME, "tx").text
            Kukmin_date=browser.find_element(By.CLASS_NAME, "t11").text
            Kukmin_news_press="KukminIlbo"
            Kukmin_main_text.append([Kukmin_title,Kukmin_main,Kukmin_news_press,Kukmin_links,Kukmin_date])
            #print(Kukmin_main_text)
        except:
            pass #daily gaewon은 에러가나지않았지만, kbs,sbs,kukmin은 오류가있음
    #print(Kukmin_main_text)
    #print(len(Kukmin_main_text))

    Kukmin_index = ["title","main","press","url","write_date"]
    Kukmin_csv=pd.DataFrame(Kukmin_main_text,columns=Kukmin_index)
    Kukmin_csv.to_csv(f'{path}/crawling/kukmin_csv_{time_stamp_format}.csv', index=False,encoding="utf-8")

    browser.quit()

#============= KBS =============

def kbsc():
    pagenum=[1,11,21,31,41,51,61,71,81,91,101]
    opts = FirefoxOptions()
    opts.add_argument("--headless")
    browser = webdriver.Firefox(options=opts)

    kbs_link=[] # KBS 본문링크만 담을 리스트
    for links in pagenum:
        kbs_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=28&mynews=1&office_type=1&office_section_code=2&news_office_checked=1056&nso=so:r,p:all,a:all&start={links}"
        browser.get(kbs_pagelink)
        all_page_lang=browser.find_elements(By.CLASS_NAME,"dsc_thumb")
        for cont in all_page_lang:
            kbs_content_link=cont.get_attribute("href")
            kbs_link.append(kbs_content_link)

    #타이틀/ 본문 / 언론사/ URL / 작성날짜

    Kbs_main_text=[]

    for kbs_links in kbs_link:
        try:
            browser.get(kbs_links) #본문페이지로 변경
            time.sleep(1)#로딩으로인해 발생하는 error를 줄이기위해 1초 대기
            kbs_title=browser.find_element(By.CLASS_NAME, "tit-s").text
            kbs_main=browser.find_element(By.ID, "cont_newstext").text
            kbs_date=browser.find_element(By.CLASS_NAME, "date").text
            kbs_news_press="KBS"
            Kbs_main_text.append([kbs_title,kbs_main,kbs_news_press,kbs_links,kbs_date])
        except:
            pass
    #print(Kbs_main_text)
    Kbs_index = ["title","main","press","url","write_date"]
    Kbs_csv=pd.DataFrame(Kbs_main_text,columns=Kbs_index)
    Kbs_csv.to_csv(f'{path}/crawling/kbs_csv_{time_stamp_format}.csv', index=False,encoding="utf-8")
    browser.quit()

#============= SBS =============

def sbsc():
    pagenum=[1,11,21,31,41,51,61,71,81,91,101]
    opts = FirefoxOptions()
    opts.add_argument("--headless")	
    browser = webdriver.Firefox(options=opts)

    sbs_link=[] 
    for links in pagenum:
        sbs_pagelink=f"https://search.naver.com/search.naver?where=news&sm=tab_pge&query=%EB%B0%98%EB%A0%A4%EB%8F%99%EB%AC%BC%20%EB%B3%B5%EC%A7%80&sort=0&photo=0&field=0&pd=0&ds=&de=&cluster_rank=22&mynews=1&office_type=1&office_section_code=2&news_office_checked=1055&nso=so:r,p:all,a:all&start={links}"
        browser.get(sbs_pagelink)
        all_page_lang=browser.find_elements(By.CLASS_NAME,"dsc_thumb")
        for cont in all_page_lang:
            sbs_content_link=cont.get_attribute("href") 
            sbs_link.append(sbs_content_link)

    # #======1-2. SBS 크롤링 / 본문내용=============

    # #타이틀/ 본문 / 입력날짜 /URL

    sbs_main_text=[]

    for sbs_links in sbs_link:
        try:
            browser.get(sbs_links) 
            time.sleep(1)
            sbs_title=browser.find_element(By.ID, "news-title").text
            sbs_main=browser.find_element(By.CLASS_NAME, "text_area").text
            sbs_date=browser.find_element(By.CSS_SELECTOR, "div.date_area span").text
            sbs_press="SBS"
            sbs_main_text.append([sbs_title,sbs_main,sbs_press,sbs_links,sbs_date])
        except:
            pass
    #print(sbs_main_text)

    sbs_index = ["title","main","press","url","write_date"]
    sbs_csv=pd.DataFrame(sbs_main_text,columns=sbs_index)
    sbs_csv.to_csv(f'{path}/crawling/sbs_csv_{time_stamp_format}.csv', index=False,encoding="utf-8")

    browser.quit()

#Airflow dag

with DAG( 
    'News_pipline', #dag id
    default_args = default_args,
    schedule = '@once',
    start_date = datetime(2023, 9, 16),
    catchup = False,
    tags = ['pip_line','spark-submit','spark']

)as dag:#그룹으로 묶지않을 Task

    Start = DummyOperator(
        task_id = "start",
        queue = "airflow-master"
    )
    
    End = DummyOperator(
        task_id = "end",
        trigger_rule = "all_success"
    )
    Master_spark = BashOperator(
        task_id = "crawling_spark_sbumit",
        bash_command =f"spark-submit --name 'crawling_spark_submit' --master yarn {path}/airflow-spark-submit-test.py",
        queue="airflow-master"
    )

    with TaskGroup(group_id='group_crawling') as Group1:#크롤링하는 Task들을 Group1이라는 이름으로 묶음(as Group1==airflow UI에서 보일 이름)
        Worker1_1 = PythonOperator(
            task_id = "daliygawon_crawling",
            python_callable = dailygaewonc,
            queue = "airflow-worker-1"
        )

        Worker1_2 = PythonOperator(
            task_id = "kbs_crawling",
            python_callable = kbsc,
            queue = "airflow-worker-1"
        )

        Worker2_1 = PythonOperator(
            task_id = "sbs_crawling",
            python_callable = sbsc,
            queue = "airflow-worker-2"
        )

        Worker2_2 = PythonOperator(
            task_id = "kukmin_crawling",
            python_callable = kukminc,
            queue = "airflow-worker-2"
        )

        Worker1_3 = SSHOperator(
            task_id ="worker1_scp",
            ssh_conn_id="ssh_master",
            command=f"""
            scp -T slave01:{path}/crawling/dailygawon_csv_{time_stamp_format}.csv {path}/crawling/
            scp -T slave01:{path}/crawling/kbs_csv_{time_stamp_format}.csv {path}/crawling/
            """,
            queue = "airflow-worker-1"
        )

        Worker2_3 = SSHOperator(
            task_id ="worker2_scp",
            ssh_conn_id="ssh_master2",
            command=f"""
            scp -T slave02:{path}/crawling/sbs_csv_{time_stamp_format}.csv {path}/crawling/
            scp -T slave02:{path}/crawling/kukmin_csv_{time_stamp_format}.csv {path}/crawling/
            """,

            queue = "airflow-worker-2"
        )

        Middle_check = DummyOperator(
            task_id = "middle_check_point",
            trigger_rule = "all_done"#바로앞 작업들이 성공일때 수행
        )



        [Worker1_1 >> Worker1_2 >> Worker1_3, Worker2_1 >> Worker2_2 >> Worker2_3] >> Middle_check
    
    Start >> Group1 >>  Master_spark >> End
    

#scp -T 옵션 = 파일을 가져오거나 내보낼경로에 이스케이프문자를 이용해야할때 규칙검사를 사용안하는 옵션. bahsrc에 등록해서 사용하거나 이스케이프 문자를 쉘에서 직접 처리하거나 -T옵션을 사용할수있다.

```

![airflow배치도](./airflow/airflow_graph.PNG)


작성한 코드는 위와같은 airflow배치도를 가진다.
spark-submit으로 spark전처리-mysql적재까지 하기때문에 airflow에는 크롤링 코드만 사용했다. (크롤링코드도 py파일로 만들수있었지만 굳이 만드는것보단 작성하여 쓰는게 낫다고 판단했다.)

또한 그룹으로 나눠 보기편하게 만들었고 트리거를 이용하여 앞의 작업이 성공하였는지 체크하고 spark전처리를 실행하는 구조를 가졌다.


spark 전처리 파일의 내부는 아래와 같다.

```bash

#**********spark 관련**********
from pyspark.sql import *
from pyspark.sql.functions import regexp_replace
from pyspark.sql.functions import col
from pyspark.context import SparkContext
from pyspark.sql.session import SparkSession
import pandas as pd
from datetime import datetime, timedelta
import csv
import re
import time

#**********기본경로,파일명선언**********
path='/home/napetservicecloud'
time_stamp_format=datetime.now().strftime('%y%m%d_%H')


#**********SPARK 세션생성**********

spark = SparkSession.builder.config("spark.jars", "mysql-connector-java-8.0.33.jar").master("yarn").getOrCreate()


#**********파일 로드**********

#daliygewon pandas read
daliygaewon_read_pd=pd.read_csv(f"{path}/crawling/dailygawon_csv_{time_stamp_format}.csv")
dailygaewon_data= spark.createDataFrame(daliygaewon_read_pd)
dailygaewon_data.createOrReplaceTempView("dailygaewon_news")
#dailygaewon_data.show()

#kbs pandas read
kbs_read_pd=pd.read_csv(f"{path}/crawling/kbs_csv_{time_stamp_format}.csv")
kbs_data= spark.createDataFrame(kbs_read_pd)
kbs_data.createOrReplaceTempView("Kbs_news")
#kbs_data.show()

#sbs pandas read
sbs_read_pd=pd.read_csv(f"{path}/crawling/sbs_csv_{time_stamp_format}.csv")
sbs_data= spark.createDataFrame(sbs_read_pd)
sbs_data.createOrReplaceTempView("sbs_news")
#sbs_data.show()

#kukmin pandas read
kukmin_read_pd=pd.read_csv(f"{path}/crawling/kukmin_csv_{time_stamp_format}.csv")
kukmin_data= spark.createDataFrame(kukmin_read_pd)
kukmin_data.createOrReplaceTempView("Kukmin_news")
#kukmin_data.show()

#**********daliygewon 가공,전처리부분**********

#main컬럼
diy_main_p=dailygaewon_data.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'승인',"").alias("write_date"))

#title컬럼
diy_title_p=diy_main_p.select(regexp_replace(col("title"),'\[[^)]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
#diy_title_p.show()

#특정문자를 삭제하고 DF에 id를 부여
dailygaewon_wordcount_x=diy_title_p.rdd.zipWithIndex().toDF()
final_dailygaewon_df_wordx=dailygaewon_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*")) #_2가 id. id가앞에오게 _2를 먼저부름
final_dailygaewon_df_wordx.show()

#dailygaewon//하둡에저장할때 파티션을 나누지않고 병합
final_dailygaewon_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv(f"/user/crawling/dailygaewon_final_url{time_stamp_format}.csv")



#**********Kukmin Ilbo 가공,전처리부분**********

#main컬럼 전처리 ex)[\n▶▲■▷▼●]'," "
kuk_main_p=kukmin_data.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'승인',"").alias("write_date"))

# title 컬럼
kuk_title_p=kuk_main_p.select(regexp_replace(col("title"),'\[[^),]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
#kuk_title_p.show()

# #id생성문
kukmin_wordcount_x=kuk_title_p.rdd.zipWithIndex().toDF()
final_kukmin_df_wordx=kukmin_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*"))
final_kukmin_df_wordx.show()

#하둡저장
final_kukmin_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv(f"/user/crawling/kukmin_final_url{time_stamp_format}.csv")

#**********KBS 가공,전처리부분**********

#main컬럼 전처리
kb_main_p=kbs_data.select(col("title"),regexp_replace(col("main"),'[▷■\n※-▶◆▼©●▲『』]'," ").alias("main"),col("press"),col("url"),regexp_replace(col("write_date"),'입력',"").alias("write_date"))

#title컬럼 전처리
kb_title_p=kb_main_p.select(regexp_replace(col("title"),'\[[^),]*\]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\([^)]*\)',"").alias("write_date"))
#kb_title_p.show()

#ID생성,CSV파일생성
kbs_wordcount_x=kb_title_p.rdd.zipWithIndex().toDF()
final_kbs_df_wordx=kbs_wordcount_x.select((col("_2")+1).alias("id"),col("_1.*"))
final_kbs_df_wordx.show()

final_kbs_df_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv(f"/user/crawling/kbs_final_url{time_stamp_format}.csv")


#**********SBS 가공,전처리부분**********


#main컬럼 전처리
sb_main_p=sbs_data.select(col("title"),regexp_replace(col("main"),'[\n▶▲■▷▼●\[\]『』©※]'," ").alias("main"),col("press"),col("url"),col("write_date"))

# title 컬럼에대하여 정규식을 사용해 여러문자 제거. []는 특수한문자이므로 이스케이프가 필요하여 앞에 \를 사용. []안에있는 문자들과 일치하면 모두 공백으로변경
sb_title_p=sb_main_p.select(regexp_replace(col("title"),'[\[,"\]+]',"").alias("title"),col("main"),col("press"),col("url"),regexp_replace(col("write_date"),'\[[^)]*\]',"").alias("write_date"))
sb_title_p.show()

#id생성,csv파일 생성

sbs_rdd_id=sb_title_p.rdd.zipWithIndex().toDF()
final_sbs_wordx=sbs_rdd_id.select((col("_2")+1).alias("id"),col("_1.*"))
final_sbs_wordx.show()
final_sbs_wordx.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv(f"/user/crawling/sbs_final_url{time_stamp_format}.csv")


#**********Mysql드라이버**********

user='db_user'
password='password'
url='jdbc:mysql://ip:port/db_name'
driver='com.mysql.cj.jdbc.Driver'
dbtable='db_table'



#**********병합을위한 ID제거**********
kbs_idx=final_kbs_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
sbs_idx=final_sbs_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
kukmin_idx=final_kukmin_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))
daily_idx=final_dailygaewon_df_wordx.select(col("title"),col("main"),col("press"),col("url"),col("write_date"))


#**********병합작업**********
result_kbs = kbs_idx.select("*").toPandas()
result_sbs = sbs_idx.select("*").toPandas()
result_kukmin = kukmin_idx.select("*").toPandas()
result_daily = daily_idx.select("*").toPandas()

concatDF=pd.concat([result_daily,result_kukmin,result_sbs,result_kbs])
news_concatdf = spark.createDataFrame(concatDF)

###ID재부여

news_concat_id=news_concatdf.rdd.zipWithIndex().toDF()
final_newsDF=news_concat_id.select((col("_2")+1).alias("id"),col("_1.*"))

#HDFS에 저장

final_newsDF.coalesce(1).write.options(header='True', delimiter=',', encoding="cp949").csv("/user/crawling/news_final_{time_stamp_format}.csv")

## Mysql에 저장

final_newsDF.write.jdbc(url,dbtable,"overwrite",properties={"driver":driver, "user":user,"password":password})


#스파크 세션종료
spark.stop()
```

![airflow_graph2](./airflow/airflow_grpah_2.PNG)

모든 Task가 정상적으로 실행되면, 위와같이 초록색으로 보인다.

