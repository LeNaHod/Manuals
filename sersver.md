# 개인 SERVER구축 공부

## WSL을 사용해서 VM없이 윈도우에서 리눅스를 사용해보자!


1.관리자 권한-CMD 실행 OR 윈도우 POWERSHELL 열어서 WSL --INSTALL (기본값으로 설치)입력

2.만약 설치된 우분투 버전을 바꾸고싶다면, WSL --INSTALL -D 설치할배포판이름 으로바꿀수있다.

3.설치할수있는 배포판의 목록을 보고싶다면 WSL -l -o (소문자 L,O)를 입력하여 설치할수있는 배포판의 목록을 알수있다.

4.설정이 끝났으면, 재부팅한다. 재부팅하면 리눅스 셸이 켜지며 리눅스 설치가 진행된다.

[WSL설치공식가이드](https://learn.microsoft.com/ko-kr/windows/wsl/install)

## 다른 SSH 서버에 접속하고 로그인하기위한 SSH설치

    SSH란?
    텔넷과같은 원격 접속방식인데, 텔넷의 보안문제를 해결한 버전이라고 볼수있다. 통신간의 암호화를 해준다.
    많은 사이트들이 SSH공개키로 인증한다. 공개키를 사용하려면 먼저 공개키를 만들어야하는데,
    저장은 주로 키들은 ~/.SSH 아래에 저장된다. 그래서 CD ~/.SSH로 키가 존재하는지, 키의 목록을 알수있다.

1.SUDO INSTALL APT OPENSSH-SERVER
2.SUDO INSTALL APT SSH-ASKPASS

위의 두 패키지를 설치후, SSH키를 만들어준다.
(ASKPASS는 OPENSSH-SERVER를 쓰기위한 X11기반 암호문구대화상자인데 그냥 두개가 세트라고생각하자)

- SSH-KEYGEN:SSH키를 만들어주는 프로그램. 보통 내장되어있고, 윈도우같은경우는 WINDOW GIT에 포함되어있다. SSH-키젠을 통해 SSH키를 새로만든다고 생각하면된다.

ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa -> SSH-KEYGEN으로 SSH키를만들건데 "공개키 타입"은 rsa, 패스워드는 비워둔다는 의미이고 -f 뒷부분은 저장하는부분으로 ~/.ssh/안의 id_rsa라는 이름으로 공개키를 만든다. 고로, **-t 뒤의 rsa는 타입이므로 바뀔수없고**, **-f의 뒷부분의 저장아이디는 바뀔수있다**

※패스워드를 비워두면 다음에 키를사용할때 암호를 묻지않는다. 저장을 ~/.ssh/id_rsa .ssh안에 id_rsa라는 이름으로 할거다. 라는 의미이다. 

그럼 그냥 id_ras 라는 파일과 id_ras.pub가 생기는데 .pub가 '공개키'이고, 이름만있는게 '개인키'이다.


### 비밀키를 가지고있는 클라이언트에 접속할때 자동인증 처리를해보자

보통은 .pub파일 내용을 복사하여 메일을 보내야하는데 귀찮으니까 autohorized_keys로 자동인증처리를하자.


cat ~/.shh/id_rsa.pub >> ~/.ssh/authorized_keys

-> id_rsa.pub라는 공개키를 cat명령어 >> 를 이용해 authorized_keys 라는 **이름으로 바꾸어 파일을 복사하였다.**  > 는 기존파일을 삭제하고 새로만들고, >>는 기존파일을 유지하고 새로만드는것

- authorized_keys : 서버에서 인증된 공개키 정보를 저장하는 파일. 권한 600. 개행 문자로 공개키를 구분하여 여러개 저장. 비밀키를 갖고 있는 클라이언트에 접속 시 자동 인증 처리.

위의 과정을거치면 ~/.ssh 파일안에는 id_rsa.pub / id_rsa / authorized_keys 이렇게 세개가 생성되게된다.(새로 설치하고 처음만든다는 기준하에)


## HADOOP설치 후 JPS실행 시 SSH 오류문제

첫번째로, ssh localhost를 실행해보자. 이게 잘실행되면 ssh키자체는 문제가없고 대부분 ssh의 이름이 이상한경우(파일안의 이름. 나 같은경우 namenode가 ssh의 이름이 누락되어있었다.)

install_setting매뉴얼을 따라 HADOOP설치 후 , start-all.sh 입력 하고 있다보면 

    localhost: ssh: connect to host localhost port 22: connection refused

위와같은 에러를 만날수있게된다.(WSL 환경) SSH-SERVER를 설치안했으면 이게 가장 큰 이유겠지만, 나같은경우는 설치는되어있는경우였기에 포트나 방화벽문제로 추정됐다. 그리고 알아본결과 이유는 SSH를 설치하고 공개키는 설정했는데 **포트**가 **열려있지않아** 발생한 오류였다. 고로 포트를 열어주면 해결되는문제인데, WSL는 과정이 좀 험난하다(SYSYEMCTL을 사용못하기때문에)

해결과정은 아래와 같다.

1 sudo apt install net-tools 설치 -> netstat -ntl 입력

정상일경우 아래와같이뜬다.

![22포트가 열려있는경우](/netstat.PNG)

비정상일 경우, 아래와같이 뜬다. (아무런정보가나오지않는다)

Active Internet connections (only servers)

Proto Recv-Q Send-Q Local Address



2 localhost: ssh: connect to host localhost port 22: Connection refused 오류해결

2-1과같이 포트오류인줄 알았는데, 포트만해결해서되는게아니였다. (포트문제였건것같지도 않다.)

2-1과 같은방식으로 포트를 변경하고 접근어드레스들을 제어할수있다는걸 알았다면, 이 오류의 근본적인문제는  **/etc/hosts.allow 와 hosts.deny의 문제**였다.

### sudo vim /etc/hosts.allow 를 입력하여 주석부분밑에 주석이 해제된상태로 접속을 허용할 IP주소를 넣는다.

![hosts.allow](/%ED%97%88%EC%9A%A9%ED%95%A0ip.PNG)

### sudo vim /etc/hosts.deny 는 포트를 막을때 사용한다. allow설정법과같이 주석해제를 하고 sshd:ALL이라고 적어놓으면 allow에서 등록한 아이피 외에서는 접속되지않는다.


![hosts.deny](/%EC%A0%9C%ED%95%9C%ED%95%98%EA%B8%B0.PNG)

**단, 하둡과 연결작업하는데 sshd:all로 막아버리면 노드들은 올라오지만, 오류가나니까 가급적 쓰지말기**
(내일물어볼수잇으면 물어보기)


## WSL설치 경로찾기

WINDOW탐색기에 경로를하나하나 입력하기 귀찮으니, \\wsl 을 쳐주면 설치된 ubunto폴더가나온다. 
이후 즐겨찾기해놓으면 편하다.

## SPARK설치와 Zeppelin 설치 실행


spark버전과 pyspark 버전은맞는데 zeppelin에서 실행이되지않는 오류를 만났다..

그래서 VM환경의 spark버전과 WSL에 설치된 SPARK버전을 봤더니, 버전이달랐다.(링크는같은데..?)

고로 설치한 spark버전을 바꾸고 pyspark도 삭제해주기로했다.

- rm spark 로 심볼릭링크삭제, wsl설치된 경로를찾아가 spark디렉터리도 삭제한다
  
- pip uninstall pyspark==3.3.1 <삭제

### spark재설치 (3.2.3버전)

wget https://dlcdn.apache.org/spark/spark-3.2.3/spark-3.2.3-bin-without-hadoop.tgz

재설치하고 경로는 미리잡아둔걸썼다. 이후 pip install pyspark==3.2.3 설치하니 zeppelin에서 인터프리터 오류가 사라졌다.

## WSL Ubuntu 좀비프로세스

    오류사항
    localhost: WARNING: nodemanager did not stop gracefully after 5 seconds: Trying to kill with kill -9

위와같은 오류를만났다. VM에서 실행할땐 한번도 만나지않았는데 하둡설치때부터 ssh키랑 이것저것 삐걱삐걱거리더니 이상하게 자주오류가난다.. 한번 날잡아서 손봐야겠다.

여튼, stop-all.sh 를 입력했을때 강제로 프로세스를 종료시키라는 메세지가떴다. 문제가되는 프로세스를 확인하고 종료해주자.


- ps -ef / kill -9 pid번호

## Mysql 삭제 & 재설치

삭제
    아래내용을 차례대로 입력
    sudo apt-get remove --purge mysql*
    sudo rm -rf /etc/mysql /var/lib/mysql
    sudo rm -rf /var/log/mysql
    sudo rm -rf /var/log/mysql.*
    sudo rm /var/lib/dpkg/info/*
    sudo apt-get autoremove
    sudo apt-get autoclean

    * dpkg -l | grep mysql
    * sudo apt-get remove --purge 파일명 
    ->위 두개는 연관된 파일이있을때 실행. ls해서 아무것도없으면 실행 x


재설치

    apt-get install mysql-server --fix-missing --fix-broken
    
    mysql을 실행해보면 잘 실행되는것을 볼수있다.


## Mysql sock 오류

>mysql -u 계정명 -p 를 입력하니 아래와같은 오류를 만남

ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (13)

그래서 mysql이 아예 동작을안하느냐? 그건아니다
앞에 **sudo**를 붙여서 실행하면 동작은된다.

하지만 매번그러기 귀찮다.. 그리고 뭔가 문제가있다는 뜻이기도하고 그래서 해결법을 알아봤는데, 대부분은

ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' **(2)**

구글링 자료는 2번자료이지 13번자료가아니다. 
별차이없는줄알고 2번으로해결했다가 재설치까지가는바람에 작성한다.


1. sock(13)

원인:mysqld.sock이 위치한 디렉터리의 접근권한이없어 sock에 접근할수가없다.

해결방법: sock파일이위치한 디렉터리의 권한을 주면 해결된다.

>sudo chmod 777 /mysqld.sock이있는 디렉터리


[참고블로그](https://blog.naver.com/islove8587/221970366883)

참고 블로그에따르면, /usr/local/mysql디렉터리의 권한문제여서 권한도 /usr/local/mysql에 주었는데,
나같은경우 local안에 mysql이없었고, sock에 접근을못하는건데 당연히 sock파일이 있는 디렉터리에 접근권한을 줘야하는게 아닌가?해서 

**mysql**을 실행하여 오류에서 알려주는 경로의 디렉터리권한을 777로 바꿔주었다.

2.sock(2)

13번과 2번의 구분을 어떻게하였냐면 13번의 경우, sock파일이 있을것으로 추정되는 위치에 접근하려면 permission denied가뜬다. 

- permission denied == 허가거부 == 권한없다

그런고로 sudo find / -name "mysql.sock"을 쳐서 안나오고, 허가거부까지뜬다면 13번일확률이 높은것같다(내 경험상..)

그 외의 경우는 sock2번이다.

원인
1. my.cnf파일이 깨졌다
2. mysql서버가 꺼져있다(비활성화상태이다)
3. mysql.sock을 못찾는경우, 경로가 이상한경우


해결방법

일단 my.cnf의 위치를찾아야한다. 

>sudo find / -name "my.cnf"

를 입력하거나, 보통은 /etc/mysql 아래에있는것같다.

>cd /etc/mysql/my.cnf 

꼭 etc아래에있지않을수도있으니 확인필수.

1.my.cnf를 찾아 sock경로를 제대로입력해주고 저장
2.서버를 종료하고 chmod -R 755 /var/lib/mysql/ -> chown -R mysql:mysql /var/lib/mysql/
2-1.서비스재시작

더 자세한 원인과 해결방법은 위의 참고블로그를 참조.


## elk설치 중 /etc/security/limit.conf파일에 대하여

DB운영하다보면 용량(디스크 용량)이 부족한 상태가될수도있고, 연계프로그램버그나 버그소스등등으로 너무많은 파일이 오픈되어, 필요한 프로그램을 원활하게 운영할수없는 경우가있다.

이런경우를 막기위해 **/etc/security/limin.conf 파일에서 리소스를 제한** 할수있다.

파일을열면 대략 domain,type,soft,hard,item, nproc,stack,nofile,core,vaule 등등의 옵션이있는데 알아보자.


|이름|역할|
|:---:|:---:|
|Domain|제한할 대상의 이름(*(all을의미),유저명,그룹명을 줄수있다(그룹의 경우 @가붙음)|
|type|제한의 강도를 정한다. soft or hard|
|type-soft|soft로 설정하면,설정한 용량이 넘어가면 **경고**메세지를 남김|
|type-hard|hard로 설정하면, 설정한 용량을 넘기지못함|
|item|제한할 항목이다. 각 core,data,seg,file,size등 여러가지가있다.|
|item-nproc|최대프로세스 갯수(KB)|
|item-stack|최대 스택 사이즈(KB)|
|itme-nofile|한번에 열수있는 최대 파일수|
|itme-core|core파일의 사이즈(KB)|
|value|제한 하고자하는 설정값|

★관리하는 DB서버의 스펙에 따라 vlaue값을 적절하게 조정하여야한다.


## GCP(구글클라우드)접속과 배포

GCP란? 구글클라우드의 약자이다. 일종의 VM인데, 구글의 거대한 컴퓨터중의 한조각을 빌려쓰는거와같다. VM환경설정을 세세하게할수있고 (사양, 네트워크,보안,방화벽 등등) 콘솔에 접속하여 손쉽게 관리할수있다는 장점이있다.

AWS<->GCP 둘다같은거다. 서비스하는곳이 다를뿐..

GCP에대한 자세한 환경설정과 설명에관해서는 아래 링크 참조

[GCP세부정보](https://velog.io/@rldnjsdlsi/GCP)

대충 요약하면 GCP 안에 GCS, DB서비스가존재한다.

- GCS: **비정형 데이터에 적합하다.**구글 클라우드 스토리지의 약자
- Colud SQL: Mysql, PostgreSQL 두가지 종류로 나뉨. VM에설치하는것보다 좋은이유는, GCP에서 유지,관리,보수 등 완전관리형 데이터베이스서비스다. 

참고자료:▼
[GCS,GCP,ColudSQL](https://techblog-history-younghunjo1.tistory.com/28) 

### 1.인스턴스생성

### 2.SSH키를통한 GCP접속

위의 로컬에서 SSH키를 만드는방식과는달리, 개인터미널을통하여 SSH키를 이용해 GCP에 접속하는 방법에대하여 알아보자. 이 외에도 여러 방법이있는데, 더 많은내용은

[GCP접속방법](https://medium.com/google-cloud-apac/gcp-ssh-%EB%A1%9C-gce-vm-%EC%A0%91%EC%86%8D%EB%B0%A9%EB%B2%95-%EC%A2%80-%EB%8D%94-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0-cb37dce39f87)

위의 링크를 참조하자!

**지금은 인스턴스에서 다른SSH키**를 통하여 접속하는방법을 알아본다.

1.SSH키 생성

로컬에서만드는 방식
>ssh-keygen -t rsa -P '' -f ~/.ssh./키아이디

보통은 이렇게만들고, 알아서 pub파일이 만들어진다.

GCP용 SSH키 만드는 방식

>ssh-keygen -t rsa -f ~/.ssh/키아이디 -C 구글아이디 -b rsa암호화방식값

보통과 달리, 패스워드부분이없어지고 구글아이디를 입력하는것이 생기고 rsa 암호화 방식을 지정하는게 생겼다.
-b 옵션같은경우는 주지않아도되지만, 연습삼아 줘보기로함!


|옵션명|내용|
|:---:|:---:|
|-t |파일타입명 ssh키의 암호화타입  rsa가 많이쓰임|
|-f|생성할 key의 이름을 설정한다.|
|-C|사용자에대한 주석입력. 여기선**인스턴스접속을 위한 유저네임으로 사용**|
|-b|type의 bytes설정한다. rsa타입의 암호화방식은 2048,4096 두가지가있는데 4096이 더 안전한키를생성할수있다.|

2.만든 key를 GCP에 등록하자

>cat ~/.ssh/키이름.pub 

나온값을 복사.

2-1.브라우저를켜서 gcp에 접속하여 아래 처럼 이동

![gcp_ssh1](/gep_ssh%ED%82%A4%EB%93%B1%EB%A1%9D1.PNG)
![gcp_ssh2](/gep_ssh%ED%82%A4%EB%93%B1%EB%A1%9D2.PNG)
![gcp_ssh3](/gcp_ssh%ED%82%A4%EB%93%B1%EB%A1%9D3.PNG)
3.ssh키를 생성한 ip주소를가진 터미널에서 gcp에 접속해보자

>ssh -i ~/.ssh/id_gep 접속하려는gcp계정명(아이디만)@접속하려는gcp의 외부ip주소

위와같이 입력하면,**SSH를통해 호스트에처음접속시 신뢰할거냐는내용이뜬다**

    The authenticity of host '접속하려는GCP계정의 외부IP (접속하려는 GCP계정의 외부IP)' can't be established.
    ECDSA key fingerprint is 키값내용
    Are you sure you want to continue connecting (yes/no/[fingerprint])?
    
    여기서 접속하려는곳의 정보가맞으면 yes를 입력해주면된다.
    
    yes를 입력하면 이후에 

    Warning: Permanently added '외부ip주소' (ECDSA) to the list of known hosts.
    Connection closed by 외부ip주소 port 포트번호

    .ssh폴더 아래 known_hosts에 접속하려는 gcp의 외부ip주소가 등록되게된다.

4.재접속해보자

>ssh -i ~/.ssh/등록된key이름 계정아이디@외부ip

처음 접속해서 hosts에 등록해놓으면 이제 위의 명령어만 치면 gcp에 접속할수있게된다!

![gcp접속성공이미지](/gcp_%EC%A0%91%EC%86%8D.PNG)

위와같은 화면이 나오면 성공. 경로가
계정이름@인스턴스명:~$ 으로바뀐다.

만약 본 터미널로 돌아가고싶으면 

>logout

입력하면 다시 개인터미널로 돌아가게된다.

### 3.기본설정 + Java를 설치해보자 

#sudo 패스워드 설정(root비밀번호)

GCE는 기본적으로 sudo 명령어에 대해 비밀번호를 요구X
But su 같은 sudo비밀번호를 요구하는 명령어에 대비하여 패스워드를 재생성해주자.

>sudo passwd
>비밀번호입력
>sudo echo '테스팅성공!'

제대로 재생성되었다면, 테스팅성공! 이라는 문구가 출력된다.

root계정으로 전환하자

>su

root@인스턴스명 :$ 

으로 바뀐다.

기존 패키지들을 업데이트해주자

>sudo apt-get update
>sudo apt-get upgrade 

#JAVA설치

GCE에는 기본적으로 JAVA가 설치되어있지않다. 고로 JAVA가 필요한 패키지,라이브러리 등등을 사용하기위해 JAVA를 설치해주어야한다.

>sudo apt-get install openjdk-11-jdk

보통 **java-8** / **java-11** 많이쓰니까 11로 설치하겠다.

설치확인을 해보자!

>java -version

    openjdk version "11.0.16" 2022-07-19
    OpenJDK Runtime Environment (build 11.0.16+8-post-Debian-1deb11u1)
    OpenJDK 64-Bit Server VM (build 11.0.16+8-post-Debian-1deb11u1, mixed mode, sharing)

    이렇게나온다면 성공. 이제 path설정만 남았다.


**PATH**설정을 해줘야하는데, JAVA가 설치되어있는곳으로 bashrc 에서 PATH를 잡아줘야한다.


Java를 설치하면 /usr/lib/jvm/ 아래에있다.

~/.bashrc 를 열어 아래내용 추가

    export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
    export PATH=$PATH:$JAVA_HOME/bin

    source ~/.bashrc 
    
    echo $JAVA_HOME | echo $PATH
    cd $JAVA_HOME | cd $PATH 
    
    두개 다 실험해보면 경로가 출력되는걸 알수있다.


### 4.GCP_VM에 MYSQL설치 ,환경설정

GCP에서 DB를 사용하려면 두가지 방법으로 사용할수있다.

1.GCP_VM 인스턴스에 직접 MYSQL과 필요한 패키지 설치

2.GCP_SQL 인스턴스를 이용하여 DB전용 인스턴스를 생성한다.

둘다 DB용으로 사용한다는점은 같은데, 어떤점이다를까?

- 비용차이. **VM인스턴스**가 더 **저렴**하고, **SQL용 인스턴스**가 더 **비쌈**
- 관리차이. VM인스턴스는 백업,스케줄러,유지,보수 등 완전관리형 데이터베이스서비스
  

**Mysql 을 설치해보자.**

※yum과 apt의 차이

- yum : 레드헷계열 설치명령어
- apt : 우분투/데비안 계열 설치명령어

각 인스턴스의 환경에 맞는 설치명령어를 사용하면된다.

#wsl이나 우분투환경에서 ssh키를 등록한 후 , 외부 ip를 통해 인스턴스에 접속한경우

>sudo apt-get update
>sudo apt-get upgrade -y
>sudo apt-get install mysql-server-8.0 -y #mysql서버설치. 설치하고싶은 버전을 선택하여 설치하면됨

> **sudo mysql_secure_installation**

위 명령어는 mysql 기본 보안설정 명령어이다. 재설치, 처음설치했을때 사용하는데 
주로 외부접속,외부저장 차단유무, root패스워드 설정, 현재 패스워드 설정 등등 보안에 관련된
설정을 y/n로 설정한다. 

root에 임시비밀번호가 걸려있을때
    처음설치나 재설치 시 root계정은 비밀번호가 none(설정되어있지않음)로 되어있다.
    secure_installation 스크립트에 임시비밀번호가 걸려있는경우가있다. 

    그럴땐 아래와같이 해결한다.
    > vi /var/log/mysqld.log  # mysqld.log 파일오픈

    temporary password 부분에 root@localhost의 비밀번호를 알 수 있다. 비밀번호를 기억하고

    다시 
    > sudo mysql_secure_installation 
    
    root부분에 임시비밀번호를 넣고 나머지 보안설정

[mysql_secure보안설정_참고자료1](https://absorbed.tistory.com/entry/mysqlsecureinstallation%EB%AA%85%EB%A0%B9%EC%96%B4%EB%A5%BC-%EC%95%8C%EA%B3%A0-%EA%B3%84%EC%8B%9C%EB%82%98%EC%9A%94)

[root에임시비밀번호걸려있을때](https://anfrhrl5555.tistory.com/126)


mysql_secure_installation 스크립트 대략적인 내용

    Enter current password for root(enter for none) : 현재 mysql패스워드. 초기설치시 none이므로 enter로 패스가능

    Set root password?[Y/n] : mysql root패스워드 설정유무. y선택시 비밀번호 입력하라고나옴


    rRemove anonymous users [Y/n] : 익명사용자 삭제 유무

    Remove test database and access to it? [Y/n] : testdb 삭제 유무.삭제해도 큰 문제 x

### mysql_secure_installation 오류

    Failed! Error: SET PASSWORD has no significance for user 'root'@'localhost' as the authentication method used doesn't store authentication data in the MySQL server. Please consider using ALTER USER instead if you want to change authentication parameters

위와같은 오류를만났다. 해당오류는 secure_installation에서 root비밀번호를 설정하려고하는데 root@localhost로 접속하여 root의 비밀번호를 생성한게아니여서 결국 **root의 비밀번호를 설정한게아니게된다.** 그래서 꼬여버리니 root로 접속하여 비밀번호를 설정하라는 안내메세지이다.

해결방법에 대하여 알아보자

    1.일단 켜놓은 터미널을 종료하거나 secure_installation에서 벗어나자

    2.터미널에 재접속하여 sudo mysql을 실행하여 mysql root계정으로 들어오자(초회한하여)

    3.ALTER USER 'root'@'localhost' identified with mysql_native_password by'새로운 비밀번호'

    4.정책에맞게 비밀번호를 변경해준다 -> 대소문자,숫자,특수문자섞어서 8자이상

    5.설정이 잘되었다면 Query OK가 뜬다. 

    6.해당계정의 권한을 설정한다

    7.flush privileges로 변경사항 적용

    8.이후 secure_installation에서 따로 설정할게 더 있다면 실행하여 비밀번호부분은 N으로 넘어가고 다른부분을 설정해주면된다.

[installation_root오류](https://seong6496.tistory.com/322)


※root계정과 작업계정은 localhost에서만 접속가능하게해놓고, 배포용계정에는 
생성,삭제(드롭x),수정,업데이트,조회 권한만줬다.

이러면 기본적인 mysql설정이 끝났다.

아직 네트워크,방화벽 설정 등이 남아있지만 그건 밑에서 설정하겠다.

-------------------------


+) ~/.bashrc 와 ~/.bash_profile 의 차이점

- ~/.bashrc : 우분투 20.04의 기본 디폴트세팅 파일
- ~/.bash_profile : 변경한 설정값만 따로 관리하기위해 새로만든 파일같음.

bash_porfile을 인스턴스를 실행할때마다 자동실행하게 하고싶으면, 아래와같은 내용을 추가해줘야한다.
    if [ -f ~/.bashrc ]; then
            . ~/.bashrc
    fi

-------------------------------

### 5.외부접속과배포

GCP에있는 Mysql에 로컬접속 외에 접속할수있게 하려고한다.

GCP Mysql에 접속하는방법은 

1.터미널
2.MysqlWorkbench

위처럼 두가지방법이있지만, 접속경로가 다를 뿐 GCP설정은 같다.

설정해보자

1.구글 클라우드 플랫폼에접속하여 방화벽을 검색하고, 방화벽규칙 만들기 클릭 

![mysql외부접속](/gep_mysql%EC%99%B8%EB%B6%80%EC%A0%91%EC%86%8D.PNG)

2.방화벽 규칙이름(영문,숫자), 설명,등 필요한거 설정해주고 **소스필터부분에 접속을허용할 ip등록**

![mysql외부접속2](/gcp_mysql%EC%99%B8%EB%B6%80%EC%A0%91%EC%86%8D2.PNG)

- 대상태그 : 접속하려는 database가있는 인스턴스명
- 소스필터 : ip유형선택가능
- 소스범위 : 접근을허용할 ip주소 등록부분. 0.0.0.0/0 으로하면 모든접속을 허용한다. 보안상 좋지않음
- 프로토콜 및 포트 : 모두허용 or tpc,udp,기타 등 선택하여 등록할수있는데 허용한 ip가 접속가능한 포트를 지정한다. 포트를 지정해놓으면 그 포트에만 접속가능(3306이면 보통 mysql만 쓸수있는것처럼)
  

3.터미널에서 GCP에접속하여 mysql.conf.d 파일을 수정해준다.

구글링을해보니, **my.ini, my.cnf**인 사람도있던데, 같은파일을열어봐도 수정해야할 부분이보이지않아서 mysql.conf.d 를보니 이 파일에 내용있었다. 

경로는 **/etc/mysql/mysql.conf.d**이다.

>sudo vi mysqld.cnf

```bash
[mysqld]
bind-address < 이부분이 수정해야할 부분이다.
```

![mysqlip설정](/gcp_mysql외부접속3.PNG)

bind-address는 기본적으로 로컬로만 접속되어있게 설정되어있다.

모든접속을허용하게 0.0.0.0 으로 바꾸면 모든설정이 끝난다.


#접속실험-터미널

>mysql -h [SERVER IP] -u 계정명 -p
EnterPassword:

접속이되면 성공이다.

#접속실험-Mysqlworkbench

1.workbench를 실행시키고 + 버튼 클릭
2.접속할 접속명,GCP외부IP, 비밀번호, 계정명,포트 입력 후, 접속하면 끝

![mysql접속](/gcp_mysql%EC%99%B8%EB%B6%80%EC%A0%91%EC%86%8D4.PNG)


만약 접속이안된다면 여러가지이유가있겠지만 제일먼저,mysql에 접속하여
>SELECT Host,User,plugin,authentication_string FROM mysql.user;

위와같이 입력하여 계정이 외부접속이가능하게 되어있는지 확인하자. localhost로되어있다면 새로 계정을 만들어줘야한다. %나 특정ip주소로.



---------------------------------

### 진행상황

airflow와 mysql연동오류 
airflow db init 하면 mysql을 찾을수없다고나옴.

/usr/local/mysql/lib안에 mysql이없음... 

->해결. airflow mysql 테이블과 계정을 제대로 연동이안되어있었던걸로 추정된다.

--------------------------------


## 번외해결방법

2-1(번외방법 포트를 바꾸는법) sudo vim /etc/ssh/ssh_config   OR sudo vim /etc/ssh/sshd_config 

나같은경우 sshd가 없는줄알고 ssh_config로 실행했는데 여튼 22번 포트가 잘열렸다..

저 config파일을 열어서 PORT22 가 주석처리되어있으면 주석처리를 해제해준 후, 저장해준다.

실험결과 sshd_config, ssh_config 두개 다 포트22 부분이 주석처리되어있었는데, 둘중하나만 풀어줘도 된다. 하지만 **sshd_config**쪽 파일이 **기본적인 설정파일인것같다.(어드레스설정 등)**


![sshd_config](/sshd_cofing.PNG)

포트가 비활성화 되어있는경우는 포트22 부분이 주석처리되어있다.


3 sudo service ssh restart

우리는 WSL환경이기때문에 systctl을 사용할수없다...그래서 service로해야하는데 

sudo service 실행할 서비스이름 실행할명령어 로 사용하면 된다. (sudo를 붙이는 편이좋다)

**sudo service 서비스이름 실행명령어** 


4 sudo service ssh status OR netstat -ntl 로 상태확인

service ssh status 를 사용하면 제대로 서비스가 실행중이면 서비스명 running 이라고 뜰것이고,
ntestat -ntl은 아까 위의 사진처럼 자세한 정보를 볼수있다. 포트정보가나오면 성공.

+추가 
포트는 정상적으로 22번으로 변경되었지만 selinux도 사용하고있어서 service 명령어가 안먹히기도하고, 또 새로운 문제가 계속나오기때문에 포트를 위와같이 포트를 변경하는 방법은, 오류가아니라 22번포트에 외부접속이 많아 다른포트로 바꾸고싶을때 사용하는것이 맞을것같다. 












