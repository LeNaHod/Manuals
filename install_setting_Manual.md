# 가상머신 하둡,스파크설치

★아래 설치에서 파일이름을 설치패키지-site.xml 로 통일하기위해, 이름이 다른 친구들을 cp작업을 통해 이름을 바꿔 파일을 복사함
## 1.업데이트&기본설치

sudo apt update

sudo apt upgrade -y

sudo apt install vim -y

**ssh서버 설치**

sudo apt install openssh-server -y
sudo apt install ssh-askpass -y

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
export SPARK_HOME=/home/na/spark
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

3. pip install pyspark == 현재스파크버전

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

#.adder/.prot 값 변경
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

%사용할인터프리터($pyspark)

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
## 8-1 Zeppelin commons버전을 낮추기(보류)

나는 commons-lang3을 설치하기싫다 하면 쓸수있는방법이다.

Interpreter설정부분에서 spark검색->edit

name:
spark.driver.extraClassPath

value:
/usr/lib/spark.jars/commons-lang-2.6.jar 

->spark와 zeppelin 어느쪽이 2.6버전인지물어보기

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
->권한 적용

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
url='jdbc:mysql://localhost:(mysql포트번호)/mysql'
driver='com.mysql.cj.jdbc.Driver'
dbtable='테이블 이름'

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
1. ~/.bashrc 에 # 에어플로우추가 

pip로 설치하기 전 bashrc에 이걸 먼저 추가해주자

↓sudo vim ~/.bashrc 에 추가 

# airflow
export PATH=$PATH:/home/계정명/.local/bin

source ~/.bashrc 로 적용!
```

## 12.elk7 / elk8설치



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




