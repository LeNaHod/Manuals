# 가상머신 하둡,스파크설치

★아래 설치에서 파일이름을 설치패키지-site.xml 로 통일하기위해, 이름이 다른 친구들을 cp작업을 통해 이름을 바꿔 파일을 복사함
## 1.업데이트&기본설치

sudo apt update

sudo apt upgrade -y

sudo apt install vim -y

sudo apt -y dist-upgrade


**ssh서버 설치**

sudo apt install openssh-server -y
sudo apt install ssh-askpass -y
***sudo apt install pdsh -y** > gcp에서 하둡설치할때 ssh 로컬작동을 위해 잠시 사용.

**net-tools&uzip 설치**

sudo apt install net-tools -y
sudo apt install unzip -y

**공개키암호화(암복호화다른거).암호안묻고 로그인하는설정**

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys


## 2.아마존 corretto=java 설치

(11버전설치할것임. 버전은자유)

wget [amazon-corretto-11-x64-linux-jdk.tar.gz](https://corretto.aws/downloads/latest/amazon-corretto-11-x64-linux-jdk.tar.gz)

- 코레토의 위치가 루트에있으면 편함 
- 루트가아닐경우 mv [이동대상] [이동할경로] (여러대상을 한번게 옮기기가능)
- mv *[이동할경로] :현재위치의 모든파일 이동

**설치 후 압축해제** 

tar -xvzf amazon-corretto-11-x64-linux-jdk.tar.gz

사용하기 쉽게 심볼릭링크(바로가기) 생성

ln -s amazon-corretto-11.0.17.8.1-linux-x64/ java

amazon[tab]-11.[tab] 누르면 / 까지 자동완성 java는  심볼릭 링크 이름

**ls -s amazon-corretto-11.0.19.7.1-linux-x64/ java** > 이건 amazon-crretto-11.0~해당 디렉터리 or 파일안에 java라는 이름을 가진 파일을 찾는듯하다.

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

**스파크 3.3.3를 다운로드받자**

wget https://dlcdn.apache.org/spark/spark-3.3.3/spark-3.3.3-bin-without-hadoop.tgz

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

sudo vi workers

#localhost 로컬호스트로 돌리고싶으면 worker설정 생략 O
worker1.name
worker2.name


3.cp spark-env.sh.template spark-env.sh

4.vim spark-env.sh 

제일 하단에 아래내용 작성

export YARN_CONF_DIR=/home/계정명/hadoop/etc/hadoop
export HADOOP_CONF_DIR=/home/계정명/hadoop/etc/hadoop
export SPARK_DIST_CLASSPATH=$(/home/계정명/hadoop/bin/hadoop classpath)
export JAVA_HOME=/home/계정명/java

export PYSPARK_PYTHON=/usr/bin/python3
export PYSPARK_DRIVER_PYTHON=/usr/bin/python3


#선택

export SPARK_HOME=/home/napetservicecloud/spark
export SPARK_CONF_DIR=$SPARK_HOME/conf
export SPARK_MASTER_WEBUI_PORT=18080(기본8080이지만 앱실행시 충돌방지를위해)

# 분산환경관련
export SPARK_WORKER_INSTANCES=2 # 워커의갯수
export SPARK_WORKER_CORES=1 #워커에 할당할 코어갯수
export SPARK_WORKER_MEMORY=1g #워커에 할당할 메모리갯수
   

5.spark-defaults.conf

★cp spark-defaults.conf.template spark-defaults.conf 를 실행하여

spark-defaults.conf 을 생성해주자.

vim spark-defaults.conf
↓

# Example: ~~ 제일하단부분에 주석없이 아래 내용추가

spark.master                              yarn

spark.eventLog.enabled  true
spark.eventLog.dir      /home/napetservicecloud/spark/logs #추후 mkdir을통해생성

6.pyspark 실행

spark/sbin/start-all.sh -> pyspark -> spark로고 확인 / master:yarn확인

7.워커등록 및 웹UI확인

# 마스터 먼저 실행
spark/sbin/start-master.sh 으로 마스터를 먼저 실행

★반드시 pyspark를 실행하기전, spark가 실행되고있어야하고 spark가실행되려면 hadoop이실행되고있어야함.

이후 생성된 마스터의 URL (spark://~~~:7077 형식)을 복사한다.

# 워커실행

spark/sbin/start-worker.sh [복사한masterURL]

단,해당작업을할때는 워커로사용하고싶은 서버로 이동해서 사용한다.
(만약같은 서버,인스턴스에서 사용시 동일한 컴퓨터 내부에서 메모리를할당하고 분산처리를하게된다.)


# spark/sbin 아래의 start-all.sh(Hadoop아님)

spark/sbin아래에있는 start-all.sh를 실행시켰을시, start-master.sh / start-workers.sh가 실행된다.
즉 저 위의 두개가 같이 실행된다.

```

![Standalone_spark_webUI](./spark분산.PNG)

<u>위는 같은서버내에서 스파크 분산처리를 실행한것. 워커가 동일한 아이피를 가지고있다.</u>

## 6. 다른 인스턴스를 spark의 워커로 실행해보자

아까 위에서 편집했던 worker.sh 파일을 보면 분명 워커로 사용하고싶은 서버의 ip나 호스트네임을 등록해놨을것이다.

하지만 당연하게도 워커로 사용하고싶은 서버나 인스턴스에도 당연히 스파크가 설치되어있어야한다.

```bash

spark/sbin 아래에 

start-worker.sh [마스터url] 은 해당 명령어를 이용하는 서버를 마스터에 워커로 등록하는것같다.

하지만, start-workers.sh 를 실행시켰더니, worker파일에 등록해놨던 서버들을 찾아간다.


```
## 7.어플리케이션 생성을 위한 pyspark설치

어플리케이션 생성 == .py생성 

spark릴리즈 버전에 맞는 pyspark설치

```bash
1. cd spark
-> 계정@ubuntu:~/spark$ 으로이동해서

2. cat RELEASE 
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

sudo mysql -u 계정명 (계정을 만들었을시)
※하지만, 해당 메뉴얼에서는 mysql을 새로설치한것으로 간주하고 계정,비밀번호가 아직 안만들어진것으로 가정

sudo mysql -u root (mysql 처음설치하고 계정없을시)

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

  아래처럼 작성해서 쉘에서 확인해도된다.
  import mysql.connector
  from pyspark.sql import SparkSession

  # spark 세션 연결
  spark = SparkSession.builder.config("spark.jars", "mysql-connector-java-8.0.26.jar") \
      .master("local").appName("PySpark_MySQL_test").getOrCreate()
      
  df = (spark
      .read
      .format("jdbc")
      .option("url", "jdbc:mysql://localhost:3306/TestDB")
      .option("driver", "com.mysql.jdbc.Driver")
      .option("dbtable", "{Table-NAME}")
      .option("user", "root").option("password", "******")
      .load())

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
->동작해서 버전이안맞으면 빨간줄로 오류가 발생. 대부분 업그레이드 오류이므로 잘 읽어보고 해당 라이브러리를 업그레이드하면됨.

아니면 버전을 직접 지정할수도있음.

pip install "apache-airflow[celery]==2.2.3" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.2.3/constraints-3.9.txt"

-> airflow버전은 2.2.3, 파이썬버전은 3.9라는것을 의미한다. 

[celery]는 airflow 분산처리를위해 워커역할을하는 비동기 작업큐이다. 주로 레딧과 레빗mq와 함께쓴다.


pip install -U requests
->설치도중 requests 업데이트 오류가나면, 이거 실행하고 에어플로우를 pip로 재설치


3. 기본DB를 MYSQL로바꾸자

sudo apt install libmysqlclient-dev -y

pip install apache-airflow-providers-mysql

mysql -u 계정명 -p -> 위의 인스톨이끝난 후 mysql접속

`에러`
   pip install mysqlclient  와 pip install apache-airflow-providers-mysql, pip install apache-airflow-providers-apache-hdfs 설치가안될때

      Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
      [end of output]

    note: This error originates from a subprocess, and is likely not a problem with pip.
    error: subprocess-exited-with-error

    위와같은 에러가 뜨며 패키지가 설치가안될때, 잘읽어보면  pkg-config를 설치해야한다고나온다.

    그러므로 pkg-config를 먼저 설치해주고 오류가났던 패키지를 다시 설치해주면 정상적으로 설치가된다.

    ★ sudo apt install default-libmysqlclient-dev pkg-config -y

(오류날시)
★ sudo apt install default-libmysqlclient-dev pkg-config -y



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
접속하려는 db정보가 다른경우 airflow부분 대체가능 localhost부분은 ip이다.


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

계정명              -       nofile         65535

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


## Zookeeper 설치

아래 주키퍼 공식 홈페이지에가서 원하는 주키퍼버전을 다운로드받는다.

[주키퍼 공식홈페이지](https://zookeeper.apache.org/) 

이후 필요한 형태로 다운로드받는다.

나는 현재 안정화된 버전이라고 안내되어있는 **zookeeper-3.7** 버전을 사용할것이다.

[주키퍼3.7](https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz)


```bash

1. 원하는 경로에 wget으로 주키퍼를 다운로드받아주기

sudo wget https://dlcdn.apache.org/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz

2. 주키퍼를 원하는 경로에 압축해제해하기

sudo tar xvzf apache-zookeeper-3.7.1-bin.tar.gz

OR 

sudo tar xvzf apache-zookeeper-3.7.1-bin.tar.gz -C /압축해제하길원하는 경로

3. 심볼릭 링크생성 OR 파일이름 바꿔주기

좀 더 편하게 쓰기위해 파일의 이름을 zookeeper라고 바꿔줄것이다.
설치한 경로에따라 mv명령어를 써서 이름을 바꾸는 동시에 파일의 위치를 원하는곳으로 옮겨줘도되고,
zookeeper라는 이름으로 심볼릭링크를 만들어줘도된다.

ln -s apache-zookeeper-3.7.1-bin/ zookeeper

or 

mv ~/경로/apache-zookeeper-3.7.1-bin/ /원하는경로/zookeeper

5. .bashrc에 zookeeper사용을위한 환경변수 추가

자바변수를 추가했다는 가정아래 주키퍼 변수만 추가

sudo vi ~/.bashrc

# zookeeper

export ZOOKEEPER=/home/계정명/zookeeper
export ZOOKEEPER_CONF_DIR=$ZOOKEEPER/conf



6. standard alone 방식(싱글노드) 으로 구축할건지 Cluster 방식(멀티노드)로 구축할지 선택

zoo.cfg 파일은 주키퍼 환경설정파일이다.
zookeeper를 처음 설치하면 '/zookeeper/conf/zoo_sample.cfg' 라는 이름으로 만들어져있다.

이 cfg파일을 zoo.cfg로 카피명령어로 바꿔준다.

sudo cp zoo_sample.cfg zoo.cfg

그리고 해당 경로에서 ls 로 확인해보면 zoo.cfg가 생긴걸알수있다.


sudo vi zoo.cfg

### zoo.cfg의 내용을 편집한다

#dataDir=/tmp/zookeeper ▼ 원래의 값을 주석처리하고 아래와같이 변경. data디렉토리와 logs디렉토리는 mkdir명령어를 통해 만들어줘야한다. 기존디렉토리 x

dataDir=/home/계정명/zookeeper/data 
dataLogsDir=/home/계정명/zookeeper/logs

#주키퍼 포트 기본값 2181. 
clientPort=2181

#아래 속성은 클라이언트가 동시에 접속할수있는 최대갯수를 설정하는것. 아래 속성을 추가하지않는다면 기본60. 0으로 설정하면 무제한

maxClientCnxns=0

#세션타임아웃 시간 설정
maxSessionTimeout=180000

#싱글노드방식은 별도의 server지정없이 위의내용만 넣어서 실행해도 된다. 하지만 카프카나 다른 어플리케이션과 연결하려면 카프카의 호스트이름과 정보를 저장해줘야한다.

server.1=HOSTNAME or ip주소:2888:3888

#멀티노드방식 

server.1=HOSTNAME or ip주소:2888:3888 <2888번부터 3888번까지라는의미
server.2=HOSTNAME or ip주소:2888:3888
server.3=HOSTNAME or ip주소:2888:3888
.
.
.
위와같은 방식으로 홀수로지정.
주키퍼는 살아있는서버가 과반수인지 아닌지를 판단해서 서비스를 지속하느냐를 결정하기때문에 한개의 클러스터는 홀수로 지정해주는것이좋다.

# 설명

zoo.cfg에서 server.1 /server.2.. 등 뒤에 1,2,3.. 이렇게 넘버링한값은 주키퍼의 myid의 설정값과 동일하게 작성해줘야한다.

나는 gcp의 비용문제로 주키퍼를 설치한 해당 인스턴스에 카프카도 같이 동작시킬것이시때문에

zoo.cfg에서 설정한 'server.1'은 해당 인스턴스에서 실행되는 카프카를 의미한다.

만약 본인이 추가로 카프카 서버를 늘리고싶다면 server.2 /server.3과같이 추가해주면된다.

# data 디렉토리 / logs디렉토리 생성

mkdir -p /home/계정명/zookeeper/data 

OR

디렉토리를 만들 위치로 이동 후 

mkdir data 

logs디렉토리도 동일한 방식으로 만들어준다.

# 사용자 권한변경

현재 접속해있는 사용자(계정)가 주키퍼 홈 디렉토리에있는 모든 파일들을 소유할수있도록 사용자 권한을 변경해줘야한다.

sudo chown -R  $USER:$USER /home/계정명/zookeeper/

파일 소유권을 부여했으면, home으로 돌아가서 ls -al로 확인

ls -al

lrwxrwxrwx 1 계정명 계정명 27 시간 zookeeper

(보통 root로되어있음, 또한 나는 zookeeper가 심볼릭 링크이기때문에 앞에 d가 안붙음)

zookeeper 디렉토리안에서도 확인작업

ls -al

drwxr-xr-x 2 계정명 계정명  4096 시간 logs

위처럼 모든 zookeeper내부의 모든 디렉토리와 파일이 현재 접속중인 사용자의 소유로 바뀐걸 볼수있다.

# 주키퍼 myid설정

아까 zoo.cfg에서 server.1,2,3 등 넘버링한 서버에대해 myid라는 파일을하나만들어서 관리한다.

각 번호에 맞는 서버가될 인스턴스에서 zookeeper아래에 data디렉토리아래에 myid를 만들것이다.

echo 1 > /home/계정명/zookeeper/data/myid

# myid값 확인

cat /home/계정명/zookeeper/data/myid

1

echo명령어로 입력한 내용이 출력되면 정상적으로 설정된것이다.

나머지 server.2 / server.3의 서버도 위와같은방법으로 2,3이 출력되면된다.


# 주키퍼 실행 & 주키퍼 실행확인

##실행
$ZOOKEEPER_HOME/bin/zkServer.sh start 

##상태확인
$ZOOKEEPER_HOME/bin/zkServer.sh status


  Using config: /home/계정명/zookeeper/bin/../conf/zoo.cfg
  Client port found: 2181. Client address: localhost. Client SSL: false.
  Mode: standalone

▲ 현재는 다른 인스턴스를 추가하지않고 하나의 인스턴스에서 주키퍼와 카프카를쓰고있기때문에 standalone으로 나온다. 만약 다른 인스턴스가추가되어 클러스터가 구성이되면 leader/follower로 나온다.

# 주키퍼 포트 확인
netstat -ntlp | grep 2181

tcp6       0      0 :::2181                 :::*                    LISTEN      3044/java

▲ 2181번 포트가 Listen이면 정상작동중.

## logs 디렉토리에 대하여

카프카나 주키퍼나 logs디렉토리는 실행정보를 저장하는것같다.
설정파일을 변경하고 아무리재시작해봐도 제대로 변경사항이 적용되지않는경우가있는데

주키퍼와 카프카의 logs 디렉토리를 초기화시키고 새로 생성하면 변경사항이적용된다.
변경사항이 적용이안될시 logs디렉토리를 열어보자.


```

## Kafka설치와 환경설정 및 실행

아파치 카프카는 주키퍼와다르게 홀수로 클러스터를 구축하지않아도된다.

아파치 카프카설치를위해 공식홈페이지에들어간다.

공식홈페이지에는 소스버전와 바이너리버전 두가지가있는데 바이너리파일을 받아준다. (버전 자유)

나는 현재 안정화된 최신버전인 Kafka3.5버전을 다운로드받을것이다.


[Kafka 공식홈페이지](https://kafka.apache.org/downloads)

[Kafka3.5](https://downloads.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz)

```bash

sudo wget https://downloads.apache.org/kafka/3.5.1/kafka_2.13-3.5.1.tgz

sudo tar xvzf kafka_2.13-3.5.1.tgz

ln -s kafka_2.13-3.5.1 kafka

# 카프카 홈 변수 설정 (/.bashrc)

sudo vi ~/.bashrc

# kafka

export KAFKA_HOME=/home/계정명/kafka
export KAFKA_HEAP_OPTS="-Xmx512m -Xms512m" #카프카가 실행될때 사용되는 용량을 지정하는 옵션. Xmx -> 최대로사용하는용량, Xms ->최소로 사용하는 용량

저장 후, 

source ~/.bashrc

환경변수가 잘 적용되었는지 확인

env | grep KAFKA

  KAFKA_HOME=/home/계정명/kafka
  KAFKA_HEAP_OPTS=-Xmx512m -Xms512m
  KAFKA_CONF_DIR=$KAFKA_HOME/config

위와같이 bashrc에 입력한대로 출력이되면 잘적용된것.

# zookeeper와같이 kafka소유권변경

카프카를 사용할, 카프카를 실행시킬 사용자(계정)에게 카프카파일의 소유권을 주기위해 파일 소유권을 지정한다.

sudo chown -R  $USER:$USER /home/계정명/kafka/

sudo chown root:root -R /home/계정명/kafka/ #현재 로그인중인 사용자말고 root로 통신하거나 실행시키고싶을때 파일소유권을 root로 변경

# 카프카 설정값 변경

server.properties 라는 파일을 변경하여 카프카 설정값을 변경해준다.

cd $KAFKA_CONF_DIR

vi server.properties

# server.properties파일을 열어서 작업.

broker.id=0

▼

broker.id=1

아까 zookeeper에서 server.1에 해당하는 인스턴스의 kafka의 broker.id를 1로 할당해준다.
만약 해당 인스턴스를 zookeeper에 server.2로 등록해놨고 myid값도 2라면, kafka의 properties의 broker.id도 2로 할당해줘야한다.

나는 현재 작업중인 인스턴스가 server.1이고 해당 인스턴스에 카프카와 주키퍼를 함께 설치하고 실행할것이다.


#listeners=PLAINTEXT://:9092

▼

listeners=PLAINTEXT://:9092 # 카프카가 '내부'에서 통신하는 주소 해당 인스턴스의 프라이빗 ip:포트 or hostname

주석을해제해주고 포트값을 변경하고싶으면 변경. 나는 디폴트값을 사용할것이다.


#advertised.listeners=PLAINTEXT://your.host.name:9092

▼

advertised.listeners=PLAINTEXT://kafka01:9092 #카프카가 '외부'에서 통신할때 '클라이언트'가 접속하기위한 주소설정

주석을해제하고 your.host.name부분에 프라이빗IP를 넣는대신 host명을 넣었다.
host명은 /etc/hosts에서 변경가능


log.dirs=/tmp/kafka-logs

▼

log.dirs=/home/계정명/kafka/logs # home이아니라 어쨌든 kafka 디렉토리의 경로를 적어두고 kafka안에 logs폴더로 지정할것이다.


zookeeper.connect=localhost:2181

▼

zookeeper.connect=kafka01:2181/gcp-kafka-cluster # 만약 kafka를 별도의 여러서버에서 실행할거라면 kafka01:주키퍼포트,kafka02:주키퍼포트,kafka03:주키퍼포트 .. 이런식으로 입력해주면된다. hostname대신 ip도 ok. 현재 내가 주키퍼에 설정한 주키퍼의 포트는 2181이므로, kafka01:2181이된것. 이부분에는 kafka의 브로커가 바라보는 zookeeper의 모든 호스트정보를 입력해주면된다.

#kafka01:2181,kafka02:2181,kafka03:2181/gcp-kafka-cluster

#뒤의 gcp-kafka-cluster는 z노드의(zookeeper) 루트 디렉토리 바로 하위에 클러스터를 구성하기위함임. 나중에 클러스터 유지보수에 용이

이후 wq!를 이용해 카프카 설정파일을 저장하고

위에서 log디렉토리설정값에 입력한 logs디렉토리를 생성한다.

mkdir -p $KAFKA_HOME/logs


# 인스턴스끼리 통신하기위해 ssh키 생성 (클러스터를 구성하기위한것)

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa

cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 키생성이 끝났으면 생성된 ssh키로 본인 인스턴스에 통신이가능한지 확인

ssh localhost


# /etc/hosts 편집

현재 작업하고있는 인스턴스의 이름은 zookeeper여서 @zookeeper라고 나오지만, 카프카와 주키퍼에 현재 인스턴스를 kafka01로 사용할것이라고 설정해놨기때문에, host 이름을 kafka01로 변경해줄것이다.

sudo vi /etc/hosts

0.0.0.0 kafka01

#더 추가할 kafka서버가있다면 아래 주석처리된것처럼 입력
#gcp내부ip(프라이빗ip) kafka02
#gcp내부ip(프라이빗ip) kafka03

저장 후 파일닫기.

sudo hostnamectl set-hostname kafka01

hostname지정

※만약 kafka02, kafka03등 현재 작업하고있는 인스턴스 외 클러스터로 묶을 kafka서버가있다면, 해당 서버에 접속해서 위의 내용처럼 hosts파일을 편집해줘야한다.

kafka02라는 인스턴스에서 hosts파일을 편집한다면 아래의 내용처럼 작성해줘야한다.

0.0.0.0 kafka02
gcp내부ip(프라이빗) kafka01
gcp내부ip(프라이빗) kafka03

★사설ip=프라이빗ip=내부아이피=192.~~~ 10.~~~

# 카프카 실행

$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties

# 데몬으로 실행
$KAFKA_HOME/bin/kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties

# 카프카 확인

netstat -nltp | grep 9092

※GCP경우 내부통신을 하기위해 방화벽 규칙에 등록해줘야한다.

카프카를 실행하기위해서는 주키퍼가 실행되고있어야한다.
카프카와 주키퍼를 데몬으로 실행하고나서 테스트용으로 토픽을 생성해보자.


# 토픽생성 확인

$KAFKA_HOME/bin/kafka-topics.sh --create --bootstrap-server kafka01:9092,kafka02:9092,kafka03:9092 --partitions 3 --replication-factor 2 --topic test

partitions = 파티션을 3개생성한다. 현재 카프카서버를 3개사용하고있으니 3군데에 저장됨

replication-factor = 나눠가질 파일의 갯수. 쉽게말하면 생성된 토픽파일을 각 서버에서 몇개씩 파일을 가질것인가.

topic = 토픽이름지정

bootstrap-server = 주키퍼를 바라보고있는 카프카서버들

```

토픽이 정상적으로 생성되었으면, 각 서버에 지정한 레플리케이션 갯수만큼 토픽파일이 생성된다.


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




