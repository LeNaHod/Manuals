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
        <value>hdfs://마스터인스턴스내부IP:포트</value>
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

### GCP Ubuntu host 추가

Slave로 사용할 인스턴스나 가상머신의 ip를 DNS로 등록해준다.

```bash


# sudo vi /etc/hosts

Slave-1 ip      Slave-1 DNS명
Slave-2 ip      Slave-2 DNS명
.
.
.
원하는 갯수까지 등록

# 내가사용한 GCP Slave설정

```

