# 가상머신 하둡,스파크설치

★아래 설치에서 파일이름을 설치패키지-site.xml 로 통일하기위해, 이름이 다른 친구들을 cp작업을 통해 이름을 바꿔 파일을 복사함
## 1.업데이트&기본설치

sudo apt update

sudo apt upgrade -y

sudo apt install vim -y


**ssh서버 설치**

sudo apt install openssh-server -y
sudo apt install ssh-askpass -y
***sudo apt install pdsh** > 해당 패키지는 'GCP'에서 하둡을 설치할때 사용한다.(로컬의 VM에서는 크게 필요X)


**공개키암호화(암복호화다른거).암호안묻고 로그인하는설정**

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.shh/id_rsa.pub >> ~/.ssh/authorized_keys


## 2.아마존 corretto=java 설치

(11버전설치할것임. 버전은자유)

wegt [tar xvzf amazon-corretto-11-x64-linux-jdk.tar.gz](https://corretto.aws/downloads/latest/amazon-corretto-11-x64-linux-jdk.tar.gz)

- 코레토의 위치가 루트에있으면 편함 
- 루트가아닐경우 mv [이동대상] [이동할경로] (여러대상을 한번게 옮기기가능)
- mv *[이동할경로] :현재위치의 모든파일 이동

**설치 후 압축해제** 

tar -xvzf amazon-corretto-11-x64-linux-jdk.tar.gz

사용하기 쉽게 심볼릭링크(바로가기) 생성

ln -s amazon-corretto-11.0.17.8.1-linux-x64/ java

amazon[tab]-11.[tab] 누르면 / 까지 자동완성 java는  심볼릭 링크 이름

  
```python
~/.bashrc 설정

sudo vim ~/.bashrc 

맨끝에 입력

# java
export JAVA_HOME=/home/계정명/java
export PATH=$PATH:$JAVA_HOME/bin

vim에서 저장하고 
터미널에서 source ~/.bashrc 입력(적용)

```

## 3.python 설치 & 설정

sudo apt install python3-pip -y

```python
sudo vim ~/.bashrc

# python
alias python=python3
```

## 4.아파치 hadoop 설치 & 설정

스파크에 하둡이 같이들어있는 패키지가있는데,

이것은 하둡실행을 위한 **라이브러리** 이므로 하둡을 데이터저장용도로 사용하고싶으면,
하둡을 따로 설치하는것을 권장.



**다운로드(hadoop3.3.4버전을 설치할것,source말고 binary를 이용하자)**

홈페이지 -> 다운로드 -> 3.3.4 바이너리 클릭-> .tar링크복사 -> 터미널에 아래코드입력

wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.4/hadoop-3.3.4.tar.gz

ls로 다운로드가 잘되어있는지 확인 후 

**압축해제 /심볼릭 링크 생성**

tar -xvzf hadoop-3.3.4.tar.gz 

ln -s hadoop-3.3.4 hadoop
->hadoop으로 심볼릭 링크생성

```python

1. vim ~/.bashrc 

# hadoop 
export HADOOP_HOME=/home/계정명/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin


source ~/.bashrc

★중요! hadoop-env.sh를 설정하러 경로를 변경해줘야함↓

2. cd $HADOOP_CONF_DIR(위에서 설정했던)

계정명@우분투:~/hadoop/etc/hadoop$ vim hadoop-env.sh

#exmport hadoop_home  ->원래주석으로 되어있던부분 
->export hadoop_home=/home/계정명/hadoop  으로 설정

#export hadoop_pid_dir=/temp ->원래주석으로 되어있던부분 
->export hadoop_pid_dir=$HADOOP_HOME/pids



#export JAVA_HOME=
->export JAVA_HOME=/home/계정명/java
# export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop 주석되어있다면 해제
# export HADOOP_OS_TYPE=${HADOOP_OS_TYPE:-$(uname -s)} 주석되어있다면 해제

3. core-site.xml 설정
vim core-site.xml (hadoop-env.sh 편집했을때랑 같은 위치)


<configuration>
	<property>
		<name>fs.defaultFS</name> 
		<value>hdfs://localhost:9000</value>
	</property>
</configuration>


4. hdfs-site.xml 설정

vim hdfs-site.xml (여기도 core-site와 위치같고, configuration태그안에 작성)

<property>
	<name>dfs.replication</name>  
	<value>1</value> 
</property>
<property>
	<name>dfs.namenode.name.dir</name>  
	<value>/home/계정명/hadoop/namenode_dir</value>
</property>
<property>
	<name>dfs.namenode.secondary.http-address</name>
	<value>localhost:9868</value>  
</property>
<property>
	<name>dfs.datanode.data.dir</name> 
	<value>/home/계정명/hadoop/datanode_dir</value>
</property> 

5. mapred-site.xml

vim mapred-site.xml (configuration태그 안에 작성)

<property>
	<name>mapreduce.framework.name</name> 
	<value>yarn</value>
</property>

6. bashrc에 HadoopUser를 추가해주자

sudo vim ~/.bashrc

# hadoopuser

export HDFS_NAMENODE_USER=계정명 
export HDFS_DATANODE_USER=계정명  
export HDFS_SECONDARYNAMENODE_USER=계정명
export YARN_RESOURCEMANAGER_USER=계정명
export YARN_NODEMANAGER_USER=계정명


저장 후, source ~/.bashrc 로 적용!

7. 네임노드 / 데이터노트 포맷과 실행

hdfs namenode -format
hdfs datanode -format
문제없이 실행되면 (common오류 대부분 오타!)

start-all.sh  실행
(start-dfs.sh + start-yarn.sh 합친것)

jps로 확인

namenode, datanode, naodemanager , secondaryNameNode, ResourceManager 가 최소


```


## 5.Spark 설치 & 설정

**스파크 3.2.2를 다운로드받자**

wget https://dlcdn.apache.org/spark/spark-3.3.1/spark-3.3.1-bin-without-hadoop.tgz

**압축해제&심볼릭 링크생성**

tar -xvzf 
ln -s spark

hadoop설치떄와 비슷한 방식으로 진행한다.

```python
sudo vim ~/.bashrc 아래내용 추가 후 source ~/.bashrc

# spark
export SPARK_HOME=/home/계정명/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin

1.cd $SPARK_HOME/conf 
1-1 계정명@ubuntu:~/spark/conf$ 경로에서 작업

2.cp workers.template workers >이부분은 hadoop과다름!

3.cp spark-env.sh.template spark-env.sh

4.vim spark-env.sh 

제일 하단에 아래내용 작성

export YARN_CONF_DIR=/home/계정명/hadoop/etc/hadoop
export HADOOP_CONF_DIR=/home/계정명/hadoop/etc/hadoop
export SPARK_DIST_CLASSPATH=$(/home/계정명/hadoop/bin/hadoop classpath)
export JAVA_HOME=/home/계정명/java

export PYSPARK_PYTHON=/usr/bin/python3
export PYSPARK_DRIVER_PYTHON=/usr/bin/python3

5.spark-defaults.conf

★cp spark-defaults.conf.template spark-defaults.conf 를 실행하여

spark-defaults.conf 을 생성해주자.

vim spark-defaults.conf
↓

# Example: ~~ 제일하단부분에 주석없이 아래 내용추가

spark.master                              yarn

6.pyspark 실행

start-all.sh -> pyspark -> spark로고 확인 / master:yarn확인
```
## 6.어플리케이션 생성을 위한 pyspark설치

어플리케이션 생성 == .py생성 

spark릴리즈 버전에 맞는 pyspark설치

```bash
1. cd spark
-> 계정@ubuntu:~/spark$ 으로이동해서

2. cat RELELASE 
->현재 spark버전확인.

3. pip install pyspark == 현재스파크버전(위의 spark설치버전에는 3.2.2로설치하는것을권장)

4. vim spark_app.py 파일 오픈

아래내용 입력

from pyspark.sql import SparkSession 

if __name__=="__main__":
    spark = SparkSession.builder.master("local").appName("myCount").getOrCreate()
    print(spark.range(1,1001).selectExpr("sum(id)").collect())

5. 실행확인 

터미널에서 입력 ↓
spark-submit spark_app.py OR python spark_app.py

오류가없으면 pyspark가 정상작동된다.

python spark_app.py 로 실행해도 되는이유는 spark와 pyspark의 버전이맞기때문이다.(pip install pyspark==3.2.2)

```
![spark-submin결과](/spark_app_py%EC%8B%A4%ED%96%89%EA%B2%B0%EA%B3%BC.PNG)

↑ spark_app.py가 제대로실행되면 중간쯤에 위와같은 결과가 나온다.

## 7.Zeppelin NootBook설치

[재플린 ver_0.10.1 다운로드링크](https://zeppelin.apache.org/download.html)

```bash

1. 위의 다운로드 링크에서 netinst버전 클릭->링크복사

2. cd 입력해서 홈으로 돌아가자

3. wget https://dlcdn.apache.org/zeppelin/zeppelin-0.10.1/zeppelin-0.10.1-bin-netinst.tgz

4. 압축해제
tar xvzf zeppelin-0.10.1-bin-netinst.tgz

5. 심볼릭 링크생성

ln -s zeppelin-0.10.1-bin-netinst zeppelin

6.sudo vim(에디터자유)~/.bashrc 오픈

# zeppelin

export ZEPPELIN_HOME=/home/계정명/zeppelin
export PATH=$PATH:$ZEPPELIN_HOME/bin

위의 내용추가

7. source ~/.bashrc 로 적용

8. zeppelin-env.sh 생성

cd $ZEPPELIN_HOME/conf

->우선 경로를 이동해준다. 계정@ubuntu:~/zeppelin/conf$

cp zeppelin-env.sh.template zeppelin-env.sh 

.template를 zeppelin-env.sh로 카피해준다.

9.zeppelin-env.sh 설정

vim zeppelin-env.sh

#19번째줄

export JAVA_HOME=/home/계정명/java

#79번째줄

export SPARK_HOME=/home/계정명/spark

#89번째줄

export HADOOP_CONF_DIR=/home/계정명/hadoop/etc/hadoop 

10. zeppelin-site.xml 생성

cp zeppelin-site.xml.template zeppelin-site.xml

11. zeppelin-site.xml 설정

vim zeppelin-site.xml

#.adder/.port 값 변경
<property>
.
.
<value>127.0.0.1</value> -><value>0.0.0.0</value>
.
.
</property>

<property>
.
.
<value>8080</value> -><value>8987</value>
.
.
</property>

위와같이 포트번호를 각 0.0.0.0 / 8987(8080은 다른곳에서도 많이쓰니까)로 변경해준다.


12. 실행확인

cd 입력해서 최상위로 이동후 아래코드실행 

zeppelin-daemon.sh start
->재플린노트북시작 

Zeppelin start                     [  OK  ]

↑정상작동이면 위와같은 메세지 출력

(zeppelin-daemon.sh strop ->종료)

※데몬이기때문에 백그라운드로 돌아감

정상작동 확인 후 , 웹브라우저를열어

localhost:포트번호(8987)

zeppelin 웹브라우저가 제대로뜨는지 확인하면된다.

```
![zeppelin](/zeppelin_home.PNG)

```bash
13. Interpreters설정

위의 이미지처럼 anonymous클릭
↓
Interpreter
↓
검색창에서 spark 검색
↓
edit (우측상단 연필모양)
↓
13-1.spark.master local[*]->yarn 으로변경

13-2.spark.submit.deployMode ->client
(싱글노드일때)

↓
하단 save 클릭


14.노트북 기본동작확인

Notebook ->Create new note로 폴더를 하나 생성하자

셸뜨는것을 확인하고 

%사용할인터프리터(%pyspark)

cardA=spark.read.formt('포맷').option("header(컬럼명유무)","t/f").option("inferSchema(스키마를 저장된대로 가져온다.컬럼타입을 알아서가져온다고 이해하면됨)","t/f").load("가져올데이터경로/파일명(*.확장자 이면 지정 확장자로끝나는거 다가져옴)").show()

※inferSchema와 Schema
->스키마 옵션을 따로 지정해주지않으면 가져올때 컬럼유형을spark가 다 String으로 설정해버린다.

infer:자동으로 컬럼타입을 spark가 가져와줌
Schema:직접 타입을 지정해서 가져올것.

ex) 

type= (name:String, age:Double, Tel:Intger)

cardAL= spark.read.schema(org.apache.spark.sql.Encoders.product[type].schema).format("csv")

정상적으로 가져오는걸 확인할수있다.
```

## 8.commons-lang3 설치

### commons-lang3란?

- java의 핵심 클래스조작에대한 지원을 제공한다. 그래서 .jar 파일이다

### 설치하는이유

- zeppelin yarn-client모드에서 spark-sql(%sql)을 사용할때 정상적으로 동작하지않기때문(CLI는 정상작동)

>spark와 zeppelin에서 사용하는 commons-lang 버전이안맞아서 나는오류이다. 

- spark : commons-lang3-3.12.0
- zeppelin : commons-lang3-3.10

```python

cd spark/jars 경로로 이동해서

ls |grep lang 
->lang을 포함하는것을 다찾아라

commons-lang3-3.12.0.jar->commons-lang3버전 확인

1. commons-lang3다운로드

★cd 로 최상위경로로 이동한 후 작업진행

https://commons.apache.org/proper/commons-lang/ 

위 링크에서 좌측 다운로드 클릭

commons-lang-2.6-bin.tar.gz 링크복사

wget https://dlcdn.apache.org//commons/lang/binaries/commons-lang-2.6-bin.tar.gz


2.압축해제

tar xvzf commons-lang-2.6.bin.tar.gz

3. 카피작업

cd commons-lang-2.6/
-> ~/commons-lang-2.6$  경로이동 후, 아래코드 입력

cp commons-lang-2.6.jar $SPARK_HOME/jars/

cd $SPARK_HOME/jars
->이동해서 ls |grep lang으로  commons-lang-2.6.jar가 있는지 확인


4. zeppelin 재시작

zeppelin-daemon.sh start

(zeppelin이 동작중이라면 /zeppelin-daemon.sh stop 후 start)

5.spark sql 동작확인

%sql
(※pyspark로 TempView생성후 사용)
select * from 테이블명

```

## 9.mysql 설치

**설치 전 실행중인sh 다 종료**

```bash

1.mysql 다운로드 & 시작확인

sudo apt install mysql-server -y
->설치

sudo service mysql start
->mysql 서비스 시작

sudo service mysql status
->mysql 상태찍기 enable인지 확인하자

1-1.mysql service 자동실행

재부팅했을때도 자동으로 켜지게 설정할것이다. 

※status상태가 이미 활성화되어있으면 생략 가능

sudo systemctl enable mysql(서비스명)

2. 계정을생성하고 자동 로그인하자

3.mysql에 원하는 계정으로 접속해보자

sudo mysql -u 계정명

4. mysql 계정추가

★mysql 설치 후 제일 처음 접속하면 root계정의 비밀번호를 재설정해줘야한다. 그래서 alter user사용해서 비밀번호를 변경해주는 작업을 진행한다. 이후 다른 계정의 비밀번호를 바꾸는데 사용할수도있다.

alter user '계정명'@'localhost' identified with mysql_native_password by 비밀번호입력;
->로컬호스트에서만 접속가능한 명시 계정명 정보 변경

create user '계정명'@'%' identified by 비밀번호입력

->어디서든 접속가능한 계정을하나 만듦 


5.만든 유저에게 권한을주자

grant all privileges on *.* to '계정명'@'localhost' with grant option;

->모든 db 및 테이블에 접근권한을 준다. 로컬에서만 접속가능,즉 *.*는 모든권한을 뜻한다.


grant all privileges on *.* to '계정명'@'%' with grant option;

->모든 db 및 테이블 접근 권한을 주고, 어디서든(로컬,리모트)접속 가능하도록 설정

★with grant option을 추가로 적어줄 경우 GRANT 명령어를 사용할 수 있는 권한도 할당된다. 

flush privileges;
->계정생성, 테이블생성, 권한 적용

6.test테이블을 생성하고 빠져나오자

use mysql
-> mysql이라는 데이터베이스를선택.use db이름

mysql 데이터베이스에 아래 테이블생성

create table test(id int, name varchar(30));
insert into test values(1, 'hadoop');
insert into test values(2, 'spark');

exit : mysql종료

```

## 10. spark와 mysql 연결

spark에서 mysql을 사용하기위해 연결 작업을 해줘야한다.
그러기위해 **mysql-connector-java**를 설치한다.

>mysql-connector-java:
mysql과 java연결해주는 친구


```bash

1.mysql-connector-java 설치

https://dev.mysql.com/downloads/

↑위 링크에서, Connector/J ->우분투 리눅스 -> 각자맞는 os버전선택 ->다운로드버튼클릭 -> 하단의 No thanks, just start my download 링크 카피 
(현재 작업우분투 버전은 20.04.5)

2.압축해제

deb파일이므로 dpkg로 진행

sudo dpkg -i mysql-connector-j_8.0.31-1ubuntu20.04_all.deb 

3.해당경로에 libintl.jar파일확인

cd /usr/share/java
ls

->위의 경로에서 libintl.jar mysql-connector-j-8.0.31.jar 파일 확인

3. spark-defaults.conf설정

위의 설정을 해놓으면 이후 zeppelin,spark 등 mysql사용가능


/usr/share/java/mysql-connector-j-8.0.31.jar
↑우선 위의경로를복사해놓고 아래 경로로이동

cd $SPARK_HOME/conf

vim spark-defaults.conf
->spark-defaults.conf 오픈

spark.jars /usr/share/java/mysql-connector-j-8.0.31.jar
->파일내용 하단에 아까 spark.jars+아까 복사해놓은 내용 붙여넣기

4.spark에서 mysql의test테이블을 불러오자

★변수에 mysql접속 정보를 저장해놓고 ,mysql에서 데이터를 가져올때 이용할것이다.

★jdbc:mysql용 포맷명. mysql데이터를 불러올때 사용한다.

user='계정명'
password='비밀번호'
url='jdbc:mysql://localhost:(mysql포트번호)/데이터베이스명'
driver='com.mysql.cj.jdbc.Driver'
dbtable='데이터베이스 안의 테이블 이름'

testDf=spark.read.format("jdbc").option("user",user).option("password",password).option("url",url).option("driver",driver).option("dbtable",dbtable).load()
->위에서 선언한 변수들을 사용하고있다.

testDf.show()

+---+------+
| id|  name|
+---+------+
|  1|hadoop|
|  2| spark|
+---+------+

아까 mysql에서 저장한 정보가 위와같이 나오면 정상적으로 spark와 mysql이 연결되었음을 알수있다.

★mysql포트번호 확인법:

1.mysql에 접속하여 글로벌변수로알아보는방법

show global variables like 'PORT';

2.접속하지않고 status로 알아보는방법

sudo service mysql status
->포트번호 찾아서 확인

```

## 11.airflow 설치

```bash
>Successful installation requires a Python 3 environment.
>Only pip installation is currently officially supported.

->python 3에서만 동작하고 pip만 공식이다.


(사전작업 / 켜져있는거 다끄기)

zeppelin-daemon.sh stop
stop-all.sh

[에어플로우링크](https://airflow.apache.org/docs/apache-airflow/stable/start.html)


1. ~/.bashrc 에 에어플로우추가

sudo vim ~/.bashrc 에 추가 

# airflow
export PATH=$PATH:/home/계정명/.local/bin

source ~/.bashrc 로 적용!

2. pip로 에어플로우를 설치해보자

pip install apache-airflow
->동작해서 버전이안맞으면 빨간줄로 오류가 발생. 대부분 업그레이드 오류이므로 잘 읽어보고 해당 라이브러리를 업그레이드하면됨

pip install -U requests
->설치도중 requests 업데이트 오류가나면, 이거 실행하고 에어플로우를 pip로 재설치

##(export AIRFLOW_HOME=~/airflow ->보류)##

3. 기본DB를 MYSQL로바꾸자

sudo apt install libmysqlclient-dev -y

pip install apache-airflow-providers-mysql

mysql -u 계정명 -p -> 위의 인스톨이끝난 후 mysql접속


2. ★계정생성과 권한설정!

create database airflow character set utf8mb4 collate utf8mb4_unicode_ci;
->utf8로되어있는 데이터베이스를 하나 생성 / 로컬에서만 사용가능

create user '계정명'@'%' identified by '비밀번호'; ->어디서든 사용가능한 계정


grant all privileges on airflow.* to 'airflow'@'localhost';
grant all privileges on airflow.* to 'airflow'@'%';


flush privileges;

airflow db init

ls

cd airflow

mkdir dags

vim airflow.cfg

#18번째

default_timezone = utc
->default_timezone = Asia/Seoul

#24번째
executor = LocalExecutor

#52번째

load_examples = True
->에어플로우가 만들어놓은 예제 노출여부 안볼거니 load_examples = False


#206번째

sql_alchemy_conn = sqlite:////home/리눅스계정명/airflow/airflow.db
->sql_alchemy_conn = mysql://airflow:비밀번호@localhost:3306/airflow 
mysql의 접속 정보이다. airflow는 sql 데이터베이스명, localhost:3306은 접속경로
접속하려는 db정보가 다른경우 airflow부분 대체가능


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

airflow db init
->위에서 설정한것을 재저장한다. 아까 위에서는 안바꿨기때문에 spllie3(기본값)이고 지금은 mysql로바꿨기때문에 새로 저장해줘야함


★airflow 계정추가

airflow users create \
--role Admin \	->권한
--username 아이디 \
--password 비밀번호 \
--firstname min \
--lastname ad \
--email admin@admin.com

★실행확인

새로운 터미널 두개를 켜서, 각각

airflow scheduler

airflow webserver

따로 실행시켜준다.

!중요!
실행시켜놓은 상태에서 -> 파이어폭스 loccalhost:8988 ->admin 계정정보입력->접속성공

★하나의 터미널로 airflow실행하기

nohup airflow scheduler &   ->53892(실행할때마다 할당되는 프로세스가다름)

nohup airflow webserver &   ->54079

↑한줄씩 따로 실행한후에 에어플로우 웹 브라우저에 접속O

## 에어플로우 프로세스 끄기

ps -ef|grep airflow ->airflow프로세스 확인

아까 nohup으로 실행시키면 뜨는 프로세스를 보고 kill해주면된다

kill 53892
kill 54079

이후 웹브라우저에 접속해 접속이 끊긴걸 확인할 수 있다.

```

## 12. VSCODE설치해보자!


[리눅스용vs](https://code.visualstudio.com/docs/setup/linux)

```python

위의 링크에서 두번째 박스에있는 내용을 하나하나 따라하면 된다. 

sudo apt-get install wget gpg

wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg

sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg

sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

rm -f packages.microsoft.gpg


sudo apt install apt-transport-https

sudo apt update

sudo apt install code # or code-insiders

------------------------------------ ▲위의 세개만 입력해도 되긴하다 아래는 자유
sudo apt update

sudo apt upgrade -y

sudo apt install code

위의 설치가 정상적으로 된다면, 터미널에서 code 라고 입력하면, vscode가 실행된다.

```

## 13.Elasticsearch7(elk7) 설치

elk란?
아래 세개를 함께묶어서 약자로 부르는 말이다.

※셋다 실시간 파이프라인 기능을가진 오픈소스 엔진이다.

- ElasticSearch : 분석,저장기능을 담당하는 엔진
- Logstash :수집부분을 담당. 서로 다른 소스의 데이터를 동적으로 통합 & 원하는 대상으로 데이터를 정규화한다.
- Kibana : 시각화,대시보드 라고생각하면된다. 위의 두 엔진을 거쳐 수집된 데이터들의 마켓을 만들어 데이터를 보기쉽게 시각화해주는 엔진이다.

Elasticsearch를 DE부분에서 어떻게사용할까?

LOG를쌓아서 LOG를KEYBANA 분석할것이다.

[Elasticsearch](https://www.elastic.co/kr/downloads/elasticsearch)

위의 링크에가서 각 os에맞는버전을 설치함
(현재는 리눅스 -> 7.17.7 버전설치)

```bash

1.설치

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.7-linux-x86_64.tar.gz


tar xvzf elasticsearch-7.17.7-linux-x86_64.tar.gz
(압축해제)

ln -s elasticsearch-7.17.7 elastic
(심볼릭 링크생성)

2.키바나 설치

https://www.elastic.co/kr/downloads/past-releases#kibana

버전은 위의 일레스틱서치와 같아야한다.

위의 링크에서 원하는버전을 누르고,다운로드누르고 링크를복사해온다.

wget https://artifacts.elastic.co/downloads/kibana/kibana-7.17.7-linux-x86_64.tar.gz


tar xvzf kibana-7.17.7-linux-x86_64.tar.gz
(압축해제)

ln -s kibana-7.17.7-linux-x86_64 kibana
(심볼릭 링크생성)

3.logstash설치

https://www.elastic.co/kr/downloads/past-releases#logstash

키바나와 같은방법으로 설치진행

wget https://artifacts.elastic.co/downloads/logstash/logstash-7.17.7-linux-x86_64.tar.gz

tar xvzf logstash-7.17.7-linux-x86_64.tar.gz
(압축해제)

ln -s logstash-7.17.7 logstash
(심볼릭 링크생성)

4.sudo vim ~/.bashrc 설정

# elk

export ELASTIC_HOME=/home/계정명/elastic
export LOGSTASH_HOME=/home/계정명/logstash
export KIBANA_HOME=/home/계정명/kibana

export PATH=$PATH:$ELASTIC_HOME/bin:$LOGSTASH_HOME/bin:$KIBANA_HOME/bin

source ~/.bashrc

5.sudo vim /etc/security/limits.conf

end of file안에 아래내용 입력


#~~~~

계정명              -       nofile         655535

#end of file


6.sudo vim /etc/sysctl.conf 파일열기

#kernel.sy~

vm.max_map_count=262144

↑파일 제일 하단에 vm~만 입력 

```

## mysql에서 데이터를 가져와보자

```bash

1.logstash랑 mysql연결

logstash-plugin install logstash-integration-jdbc

2. vim ~/logstash/test.conf


input {
    jdbc {
      jdbc_driver_library => "/usr/share/java/mysql-connector-j-8.0.31.jar"
      jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
      jdbc_connection_string => "jdbc:mysql://localhost:3306/mysql"
      jdbc_user => "root"
      jdbc_password => "1234"
      statement => "SELECT * from test"
      schedule => "* * * * *"
    }
}

filter {

}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "test"
    document_id => "%{id}"
  }
}

↑test.conf 안에 위의 내용 붙여넣기!


3.

★터미널 3개켜서 진행

3-1.기존터미널

elasticsearch 

->elasticsearch 실행 
정상실행되면[2022-11-21T14:36:45,688][INFO ][o.e.c.m.MetadataMappingService] [ubuntu] [.kibana_7.17.7_001/eQaBG~~(생략)] update_mapping [_doc] 이런식으로뜬다.


3-2.logstash -f ~/logstash/test.conf

->1분에 한번씩 select * from test가 출력된다.

3-3.kibana

-> 위의 세개 다 입력후에 아래 localhost에 접속해서 확인

★터미널 한개로 켜기(airflow와 동일한방법)

elasticsearch -d (데몬으로실행, 백그라운드로 돌아간다)

nohup kibana &

nohup logstash -f ~/logstash/test.conf &

각 아래 로컬호스트에가서 정상작동되는지  확인



## 14.셀레니움과 크롬,크롬드라이버 설치하기

우분투에서 크롤링 해오기위해 셀레니움과 크롬드라이버를 설치하자.
```bash

1.아래내용을 차례대로 입력

wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add - 

sudo sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' 

sudo apt-get update 

sudo apt-get install google-chrome-stable 


정상적으로 설치 시, 

google-chrome --version  버전을 입력했을때

Google Chrome 108.0.5359.71 

설치 시점에서 가장 최신버전의 크롬이 설치되는걸 볼수있다.

2. 크롬드라이버 설치

https://sites.google.com/a/chromium.org/chromedriver/

현재버전에 맞는 버전을 선택하여 다운로드를 클릭

Index of /버전 /  < 이런페이지로 이동되면,

운영체제에맞는 (지금은 리눅스설치니까 리눅스64)링크를 복사하여 wget으로 설치

wget https://chromedriver.storage.googleapis.com/108.0.5359.71/chromedriver_linux64.zip

라이브러리 설치

sudo pip install xlrd (셀레니움이아니라 엑셀파일을 읽기위한 패키지)

sudo apt-get install xvfb  (가상디스플레이 버퍼가없는 리눅스환경에서 셀레니움을 사용하기위한 패키지)

sudo pip install pyvirtualdisplay (xvfb를 사용하게 도와주는 패키지)

3.셀레니움 설치

pip install selenium
```

## 파이어폭스로 셀레니움쓰기

geckodriver이용


pip install webdriver-manager 설치하고

https://github.com/mozilla/geckodriver/releases 가서 다운로드받아준다.

echo $PATH 를 입력해서 나오는 경로가 코드에서 사용할 geckodriver 의 경로이다.
(/usr/bin추천)

셀레니움이있고, geckoderiver를 다운받았으면 바로 사용하면된다.

★ 단, --headless 옵션을줘야함 (리눅스같은경우는 디스플레이문제때문에)

EX)

from selenium.webdriver import FirefoxOptions

opts = FirefoxOptions()
opts.add_argument("--headless")		>GUI디스플레이 부분
browser = webdriver.Firefox(service=webdriver_service, options=opts)

browser.get(가져올url주소)

변수 = browser.find_elements(By.조건 , 상세조건 )

find_elements는 리스트를 반환하고 없으면 빈 리스트를반환한다. 그냥 element는 한개만.

## DBeaver 설치

DBeaver란? 무료 DB중하나로써 많은종류의 DB를 지원한다. JAVA기반으로 작동한다는 특징이있으며
오픈소스이다.
실행할수있는 환경은 윈도우,리눅스,MAC 에서 구동시킬수있다.

더 자세한내용은 [DBeaver홈페이지](https://dbeaver.io/)에가서 확인해보자 

나는 DBeaver를 추천받아 많은 종류의 DB를 지원한다는것을 보고 편리할것같아 설치하였다.

1.위에 링크로 연결되어있는 공식 홈페이지에들어가 Download 클릭

![디비버 공식홈페이지](./DBeaver.PNG)

위와같은 페이지가나오면 DBeaver Community에서
자신에게 맞는 OS를선택해 파일을 다운로드한다.
나는 window에 설치할것이라서 install파일로 받았다.

2.언어와 설치옵션을 선택하자

▼ install 파일을 실행시키면 아래와같은 화면이뜬다.

![디비버 언어 설치](./DBeaver_2.PNG)

영어,한국어,일본어 하고싶은대로 선택하고 다음을누르자


![디비버 설정](./DBeaver_3.PNG)

모든유저가 사용할수있도록할것인지 특정 유저만 사용할수있게할것인지 선택.

![디비버 설정2](./DBeaver_4.PNG)

여기서 필수는 기본으로 선택되어있는 위의 두개이다.(Java기반으로 개발되었으니 당연히 Java가필요함,맨위의DBeaver는 우리가 원래설치하려던것이니까 당연)

그럼 밑에두개는 뭘까해서 찾아봤더니

- Rest Settings : 전에 DBeaver가 깔려있었을수도있으니 초기화하는것
- Associate .SQL files : SQL파일들을 실행시키거나 오픈하면 기본적으로 DBeaver를 실행시킴

이렇게 이해하면될것같다. 나는 두개 다 필요없으니 패스했다.

![디비버 설치3](./DBeaver_5.PNG)

마지막으로 설치할 저장소를 선택해주면, 알아서 설치가되고 끝난다.




-----

2022-11-15 - 2022-11-16 설치오류:

jps시 ResourceManager만 나옴

↓↓↓↓↓↓

2022-11-17 
오류해결!

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorize_keys

↓↓↓↓↓↓

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 바꿔서 재실행

authorized_keys 에 d가빠져서 오타가되어서,제대로 ssh에 문제가생김.

>error=localhost: 계정명@localhost: Permission denied (publickey,password).

--------




