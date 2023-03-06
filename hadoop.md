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
                <name>dfs.namenode.http.address</name>
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

#
<property>
    <name>dfs.namenode.http.address</name>
    <value>GCP마스터인스턴스내부IP:포트</value> #포트자유
</property>
.
.
.
이하 세컨더리 네임노드의 주소도 동일하게 설정

```

**GCP말고 로컬PC나 가상머신에서 localhost로 설정하고 hdfs-site.xml에 namenode-address를 따로 설정안해줬어도 브라우저 접속이**
**잘되었는데, GCP에서는 namenode의 core-site.xml포트와 hdfs-site.xml를 다르게 설정하고 브라우저에 외부ip를 입력해줘야 접속할수있다!**


