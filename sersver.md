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

※패스워드를 비워두면 다음에 키를사용할때 암호를 묻지않는다. 저장을 ~/.ssh/id_ras .ssh안에 id_ras라는 이름으로 할거다. 라는 의미이다. 

그럼 그냥 id_ras 라는 파일과 id_ras.pub가 생기는데 .pub가 '공개키'이고, 이름만있는게 '개인키'이다.


### 비밀키를 가지고있는 클라이언트에 접속할때 자동인증 처리를해보자

보통은 .pub파일 내용을 복사하여 메일을 보내야하는데 귀찮으니까 autohorized_keys로 자동인증처리를하자.


cat ~/.shh/id_rsa.pub >> ~/.ssh/authorized_keys

-> id_rsa.pub라는 공개키를 cat명령어 >> 를 이용해 authorized_keys 라는 **이름으로 바꾸어 파일을 복사하였다.**  > 는 기존파일을 삭제하고 새로만들고, >>는 기존파일을 유지하고 새로만드는것

- authorized_keys : 서버에서 인증된 공개키 정보를 저장하는 파일. 권한 600. 개행 문자로 공개키를 구분하여 여러개 저장. 비밀키를 갖고 있는 클라이언트에 접속 시 자동 인증 처리.

위의 과정을거치면 ~/.ssh 파일안에는 id_rsa.pub / id_rsa / authorized_keys 이렇게 세개가 생성되게된다.(새로 설치하고 처음만든다는 기준하에)


## HADOOP설치 후 JPS실행 시 SSH 오류문제

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



--------------------------------



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












