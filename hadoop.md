# Hadoop $ HDFS

## Hadoop: 대용량 자료를 처리하기위한 프레임워크. 여러 모듈을 포함하고있다.

- HDFS(스토리지 계층)
- Hadoop Common
- YARN(자원관리계층)
- MAPREDUCE(처리계층)

## HDFS

자바언어로 작성된 분산 확장 파일시스템이다.
대용량 데이터를 분산저장하기위해 설계되었으며 데이터를 여러 노드에 중복 저장한다.

## Master 노드와 Slave 노드

Master노드와 Slave노드로 나눠서 운영시, hdfs / dfs 명령어는 Master노드에서 내린다.
(어떤노드들을 어떻게 사용할지는 자유겠지만)

- Master node: Master노드에서 hdfs 명령어를이용해 데이터를 적재, 관리하는 용도로 사용한다.
- Slave node: 주로 대용량 데이터를 적재하는용도사용.


# Hadoop 분산시스템 구축

GCP 를 이용하여 Master - Slave 를 별개의 인스턴스에 생성하여 데이터가 분산되어 저장되게할것임

## Hadoop 설정

**통신을 GCP의 일반 유저계정으로할게아니라 ROOT계정으로 진행할것이라 ROOT계정에서 SSH키를 만들고 배포하고, 설정해야한다. ROOT계정에서 SSH호스트명 을 입력했을시, 다른 인스턴스로 접속이되면 OK, 만약 구축을 진행하려는계정이 ROOT가아니고 일반 유저계정일때는 유저명@master or slave01 등으로 표시된상태에서 ssh키와 DNS를 설정해줘야한다.**

단, ~.xml류나 env.sh 류 설정과 다운로드는 일반계정으로해도 무관하다.


### Hadoop간의 통신할 계정설정

```bash

#난 root로 통신할거니까 root로함

>sudo chown root:root -R /home/계정명/hadoop-3.3.4

```
### Haoop ssh교환 및 네트워크 설정

```bash

>su
passwd:
>root@master or root@slave01 ... 

★단, SSH키를만들때 패스워드가 없는 키를 만들어야한다.

GCP의 일반계정말고 ROOT계정으로 통신할것이므로, ROOT@인스턴스/호스트명 인 상태에서 한다.

### SSH키 생성

1.ssh-keygen -t rsa -m pem -f ~/.ssh/키이름 -P '' # -pem 은 선택사항.추후 confing에서 사용할 ssh키를 입력할때, 반드시 pem키가 아니여도 괜찮다.

2.생성된 ssh 퍼블릭키를 authorized_keys와 GCP SSH메타 데이터에 등록(SSH-COPY를 이용해도 OK)

authorized_keys에 등록
>cat ~/.ssh/slave01-root.pub >> ~/.ssh/authorized_keys

GCP 콘솔의 SSH키 메타데이터등록(server.md참조) / ssh키를 통신할 슬레이브 인스턴스에등록
>ssh-copy-id root@master # pub키들을 전송한다.
ssh-copy-id root@slave01
ssh-copy-id root@slave02
ssh-copy-id root@slvae03

3.authorized_keys 권한변경

>chmod 0600 authorized_keys 

4. config파일 생성 및 권한변경

등록한 ssh키가 있는 위치(~/.ssh)에서 config파일 생성. 해당작업은 각 인스턴스에서 모두진행.

>vi config

HOST master(ssh master 이런식으로 쓰이기때문에 이름은 자유)
        HostName gcp외부 ip
        User 리눅스유저명
        IdentityFile /home/계정명(ex:root)/.ssh/ssh키

HOST slave01
        HostName gcp외부ip
        User   리눅스유저명
        IdentityFile /home/계정명/.ssh/ssh키

>chmod 440 config

5. 접속 테스트

>ssh master or slave01 or slave02...

root@master or slave01 으로나오면 성공.



### hostname 설정 확인 및 변경

>hostname # hostname확인. GCP에서는 기본적으로 유저명@인스턴스명 으로 초기설정이되어있는데, 이것은 호스트명이 인스턴스명으로 등록되어있는것이다. 그래서 hostname명령어를 치면 인스턴스명이나오고, hostname을 변경해주면ㅡ '유저명@바꾼hostname명' 으로 나온다.

>sudo hsotnamectl set-hostname 해당인스턴스의 바꿀DNS이름(인스턴스당 1)

sudo hostnamectl set-hostname master # 각 인스턴스마다
sudo hostnamectl set-hostname slave01
sudo hostnamectl set-hostname slave02
sudo hostnamectl set-hostname slave03....

## 호스트이름으로 통신이 가능하게 하자.

- 각 인스턴스 마다 master, slave01 이라는 이름으로 통신이 가능하도록 설정을 한다.


# sudo vi /etc/hosts

Slave-1 ip      Slave-1 DNS명
Slave-2 ip      Slave-2 DNS명
.
원하는 갯수까지 등록

# 내가사용한 GCP Slave설정

GCP_MasterInstance 내부ip       Master
GCP_Slave-1 내부ip      Slave01
GCP_Slave-2 내부ip      Slave02

```

hadoop-env나 기타 실행경로 지정, 기타옵션은 해당 레포지터리의 [Install_setting_Manual]()참고.

### core-site.xml

- Hadoop 웹사이트의 설정을하는 파일. 하둡에 저장된 데이터/디렉터리를 간단하게 보여주고 정보도 간단하게보여준다.
- 해당파일을 설정하지않았을시, 디폴트파일의 설정을 따른다.

```bash

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://server-ip:9000</value> #포트는 자유. 
    </property>
</configuration>


# 내가사용한 GCP용 설정

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://마스터인스턴스내부IP:포트</value>  #지정한 마스터ip의포트로 통신
    </property>
</configuration>

```

### hdfs-site.xml

- dfs.replication : hadoop 실행시 복제파일 갯수 지정 
- dfs.namenode.name.dir / dfs.namenode.checkpoint.dir / dfs.datanode.data.dir : ~.dir종류는 주로 노드들의 정보를 저장하는 파일을 지정해줌
- dfs.namenode.http.address / dfs.secondary.http.address: 현재 실행되는 하둡의 실행정보들을 보여주는 주소지정

```bash
<configuration>
       <property>
                <name>dfs.replication</name> # 1개:가상분산모드 3개:완전분산
                <value>3</value> 
        </property>
        <property>
                <name>dfs.permissions.enabled</name>
                <value>false</value>
        </property>
        <property>
                <name>dfs.webhdfs.enabled</name>
                <value>true</value>
        </property>
        <property>
                <name>dfs.namenode.http.address</name> #http-address도가능
                <value>server-1:50070</value> #포트자유
        </property>
        <property>
                <name>dfs.secondary.http.address</name>
                <value>server-1:50090</value> #포트자유
        </property>
        #경로는 자유
        <property> 
                <name>dfs.namenode.name.dir</name>
                <value>경로</value>
        </property>

        <property>
                <name>dfs.namenode.checkpoint.dir</name>
                <value>경로</value>
        </property>

        <property>
                <name>dfs.datanode.data.dir</name>
                <value>경로</value>
        </property>
</configuration>

#내가사용한 hadoop 네임노드 페이지 설정(네임노드는 데이터노드들을 관리하는 어드민같은것.)
<property>
    <name>dfs.namenode.http.address</name> #이 옵션이 네임노드를 볼수있는 주소 설정하는것.
    <value>GCP마스터인스턴스내부IP:포트</value> #포트자유
</property>
.
.
.
이하 세컨더리 네임노드의 주소도 동일하게 설정

※세컨더리 네임노드: 주기적으로 주 네임노드에 edit로그를 요청하여 fsimage를 업데이트하고 네임노드에 복사한다. 안전한 hadoop운영을 위해서 세컨더리 네임노드를 별도의 환경에 만드는게좋다.

체크포인트는 세컨더리 네임노드가 fsimage와 edit파일을 통합하는 작업을 의미함.

#hadoop sub 인스턴스 환경

#hdfs-site.xml 레플리카 1개 / address port 동일

<property>
        <name>dfs.replication</name> # 1개:가상분산모드 3개:완전분산
        <value>1</value> 
</property>

```

**GCP말고 로컬PC나 가상머신에서 localhost로 설정하고 hdfs-site.xml에 namenode-address를 따로 설정안해줬어도 브라우저 접속이**
**잘되었는데, GCP에서는 namenode의 core-site.xml포트와 hdfs-site.xml를 다르게 설정하고 브라우저에 외부ip를 입력해줘야 접속할수있다!**

### 내가사용한 site.xml류. hadoop-env.sh 상세설정(모든 인스턴스에 설정동일)

★ 마스터인스턴스외부ip(namenode):9870 -> 하둡 마스터 네임노드의 정보를볼수있다.
★ 슬레이브인스턴스외부ip(SecondaryNameNode):secondary.http.address 에서 밸류값으로준 포트 -> 세컨더리 네임노드정보를 GUI로 출력한다.

```bash

##bashrc 설정

# HADOOP
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin



### 나는 root계정으로 통신할것이기때문에 root로 지정. 만약 원하는 계정이있다면 root자리에 계정명입력
### ex) export HDFS_NAMENODE_USER="계정명"

# HADOOP USER
export HDFS_NAMENODE_USER="root"
export HDFS_DATANODE_USER="root"
export HDFS_SECONDARYNAMENODE_USER="root"
export YARN_RESOURCEMANAGER_USER="root"
export YARN_NODEMANAGER_USER="root"

## hadoop-env.sh 

>cd $HADOOP_CONF_DIR 
>sudo vi hadoop-env.sh

#exmport hadoop_home  ->원래주석으로 되어있던부분 
->export hadoop_home=/home/계정명/hadoop  으로 설정

###(계정=hadoop이 설치되어있는 경로의 상위. 만약 hadoop이 home아래 loc라는 폴더에 설치되어있으면 /home/loc/hadoop 으로작성)

#export hadoop_pid_dir=/temp ->원래주석으로 되어있던부분 
->export hadoop_pid_dir=$HADOOP_HOME/pids



#export JAVA_HOME=
->export JAVA_HOME=/home/계정명/java
# export HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop 주석되어있다면 해제
# export HADOOP_OS_TYPE=${HADOOP_OS_TYPE:-$(uname -s)} 주석되어있다면 해제


## sudo vim core-site.xml
### 모든 데이터노드들이 master에 9000번 포트로 통신을 한다.

<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>


## sudo vim hdfs-site.xml

<configuration>
    <property>
        <name>dfs.replication</name>
        <value>2</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///hdfs_dir/namenode</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///hdfs_dir/datanode</value>
    </property>

    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>slave01:50090</value>
    </property>
</configuration>

### sudo vim yarn-site.xml


<configuration>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>file:///hdfs_dir/yarn/local</value>
    </property>

    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>file:///hdfs_dir/yarn/logs</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>master</value>
    </property>
</configuration>


### sudo vim mapred-site.xml

<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

### 마스터 인스턴스에 슬레이브 인스턴스들을 등록해주자

```bash
>cd $HADOOP_CONF_DIR (~/.bshrc에 등록해놓은 하둡 경로)
>vi workers (마스터인스턴스에 워커들을 등록해주는과정)

slave01
slave02....
```

만약 슬레이브 인스턴스들을 늘리고싶다면, 인스턴스를 생성해서 마스터 인스턴스의 /etc/hadoop/workers에 추가해주자.(기본값 localhost지우고, 슬레이브들을 각 노드들에 등록해줘야한다. localhost가 남아있으면 접근오류남.)

### 노드들을 포맷해주자. namenode와 secondary_nameNode에서 실행

master와 secondary_nameNode에서

```bash

#hadoop-3.3.4 / hadoop이 설치되어있는 경로에서 실행

/home/계정명/hadoop-3.3.4/bin/hdfs namenode -format /hdfs_dir

# 심볼릭한 경우
/home/계정명/hadoop/bin/hdfs namenode -format /hdfs_dir

```

## workerNode, DataNode에서

```bash

/home/계정명/hadoop-3.3.4/bin/hdfs datanode -format /hdfs_dir/

# 심볼릭한 경우
/home/계정명/hadoop/bin/hdfs datanode -format /hdfs_dir

```


![하둡분산_1](/GCP%ED%95%98%EB%91%A1%EC%84%A4%EC%B9%98/%ED%95%98%EB%91%A1%EB%B6%84%EC%82%B0.PNG)

마스터:9870으로 접속한 마스터인스턴스의 네임노드 GUI. 슬레이브 인스턴스가 스토리지로 등록되어있는것을 확인할수있다. (마스터인스턴스는 용량을 20GB로, 슬레이브는 15GB로 설정해놓았다. 사진을보면 확인가능)

![하둡분산_마스터인스턴스_JPS](/GCP%ED%95%98%EB%91%A1%EC%84%A4%EC%B9%98/%ED%95%98%EB%91%A1%EB%B6%84%EC%82%B0_%EB%A7%88%EC%8A%A4%ED%84%B0%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4.PNG)

마스터 인스턴스에서 start-all.sh를 입력(하둡실행)하고나서 jps명령어를 통해 실행되는 jps목록들을 확인했다. 마스터에서 실행되는 네임노드가 잘 실행되고있다.

```bash
>start-all.sh 

Starting namenodes on [master]
Starting datanodes
Starting secondary namenodes [slave01]
Starting resourcemanager
Starting nodemanagers
```
![GCP분산구축_슬레이브](/GCP%ED%95%98%EB%91%A1%EC%84%A4%EC%B9%98/%ED%95%98%EB%91%A1%EB%B6%84%EC%82%B0_%EC%8A%AC%EB%A0%88%EC%9D%B4%EB%B8%8C_GUI.PNG)

hdfs-site.xml 의 dfs.secondary.http.address 부분에서 지정해준 주소와 포트로 접속하여 세컨더리 네임노드 동작,정보를 볼수있다.

![GCP분산구축_슬레이브_인스턴스](/GCP%ED%95%98%EB%91%A1%EC%84%A4%EC%B9%98/%ED%95%98%EB%91%A1%EB%B6%84%EC%82%B0_%EC%8A%AC%EB%A0%88%EC%9D%B4%EB%B8%8C%EC%9D%B8%EC%8A%A4%ED%84%B4%EC%8A%A4.PNG)

슬레이브 인스턴스에서는 START-ALL.SH로 실행X / 마스터인스턴스에서만 실행.

이후, 슬레이브 인스턴스를 하나 더 만들어서 1개의 마스터인스턴스와 두개의 슬레이브로 하둡멀티클러스터를 구축하였다.

