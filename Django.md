### Test 인스턴스를 한개 더 생성하자

도커 이미지로 배포하여 도커 컨테이너로 프로젝트가 정상적으로 실행되는지 비교하기위해,
Django서비스만 운영할 Test인스턴스를 한개 더 생성하여 비교해볼것이다.

사양은 Master 인스턴스와 같게 설정한 후, 동일하게 고정ip를 발급받을것이다.

로컬에있는 파일을 올리는방법은 여러가지가있겠지만, 나는 GCP콘솔 페이지에 접속해 SSH를눌러 인스턴스에 접속한 후, 파일업로드를 눌러 Django프로젝트 파일을 업로드했다.

이후 unzip패키지를 설치해 zip파일의 압축을 해제해줬다.

```bash
#패키지설치하기
>sudo apt install unzip -y

#압축해제
>unzip [파일명.zip]

#압축하기
>zip [파일명.zip] [폴더명]

```

## 로컬 Vscode에 Django Test용 인스턴스를 연결

Django 프로젝트를 사용할때 파일내부구조를 Vscode를 통하면 보기 쉬우므로 로컬의 Vscode와 연결하여 
사용해봄.

- ctrl+shift+p : 가상환경을 vs코드에서 자주설정해줄것기이기때문에 단축기를 익혀두자



**[참고사이트]**

[VSCODE GCP인스턴스에 연결하기](https://wooiljeong.github.io/server/gce-vscode/)

```bash

우선 로컬의 ssh키와 테스트인스턴스의 키를 교환해준다.

#vs코드가 wsl가아닌 로컬의 윈도우에 설치되어있으니, 윈도우의 ssh키를 따로만들어서 GCP인스턴스와 교환해준다.

# cmd <-> 리눅스 명령어

cmd에 alias 처럼 리눅스 명령어로 바꾸어 쓸수있다.

dir or tree /f = ls
type = cat
rd or rdir =rm -r /rm -rf 
del = rm
cd = pwd or cd

# del로 삭제된건 복구x

#vscode에 remode ssh를 설치하고 연결하자

이후 vs code를 열고, 왼쪽 하단의 EXTENSTIONS에 들어가서 필요한 패키지를 받아주자.
Remote SSH 플러그인을 검색하여 설치한다.

설치가 끝나면 좌측 하단에 새로생긴 Remote SSH플러그인 아이콘을 클릭하여, +버튼을 눌러 연결할 GCP인스턴스의 정보를입력해주자.

여기서, ssh키가 있는 위치에 config파일을 만들어서

>ssh -i ssh개인키 gcp계정@gcp인스턴스외부ip

위와 같은 형식대신

>ssh gcp1

과 같은 형식으로도 연결할수있다.

연결할 gcp의 정보를 모두입력하고 ssh 개인키를 통해 접속하면,
초기화면에 OS를 선택하는 바가 생기고, 연결한 인스턴스의 OS환경에 맞춰 선택해주면 연결이 끝난다.
```
![성공적으로ssh연결이추가된모습](./Django_gcp/gcp_django_vscode.PNG)

## Test인스턴스에서 업로드한 서비스를 실행시켜보자

서비스를 실행시키기전에 환경을 만들어준다.
Django프로젝트안에 같이들어있는 pip 패키지 설치리스트 requirements.txt를 설치하자

```bash

pip install -r requirements.txt

# 필요 pip중에 mysql-client가 있는데, mysql-config not found를 만났다. 해당 오류는 mysql-client 패키지를 설치하기전에
# 헤더파일을 설치해야한다고 나와있다. 링크참조
# https://pypi.org/project/mysqlclient/

위의 링크에서 안내해준대로 아래 헤더패키지를 먼저 설치함

>sudo apt-get install python3-dev default-libmysqlclient-dev build-essential

이후 mysql-client를 설치한다

>pip install mysql-client

```

