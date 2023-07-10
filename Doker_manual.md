# Docker

Docker Version : 5:20.10.22

## Preview
Doker로 펫 프로젝트 Django앱을 Doker이미지로 생성하여, GCP로 운영,배포

- 로컬개발환경에서 개발이 완료된 이미지를 Doker이미지로 말아서 GCP로 배포할것이다.
- Djnago+Nginx를 Doker를 통하여 배포

--------

## Doker 이미지와 컨테이너

Docker를 사용하기전, 도커 이미지와 컨테이너, 레이어에 대해 간단하게 리뷰한다.
Docker를 사용하려면 이미지를 생성하는쪽과 이미지를받아 컨테이너를 실행하는쪽 둘다 도커가설치되어있어야한다.


### 도커이미지?

도커 이미지는 '컨테이너 생성' 을 위한 여러 실행파일들을 묶은것.
컨테이너의 생성 및 실행을 위해 모든 파일과 환경값(우분투면 우분투 환경값, 오라클은 오라클 환경값 등)을 묶어놓은것이라고 할수있다.

그래서 이러한 도커이미지파일들은 해당 프로젝트가 운영되던 환경의 실행파일,port등 모두 포함되므로 nnMB~ nnGB로, 꽤 무겁지만 가상머신이미지보단 가벼운편이다.
이미지의 특징으로는

- 하나의 이미지로 여러개의 컨테이너를 생성할수있다.
- 이미지->컨테이너 생성,실행 이므로 컨테이너를 삭제해도 이미지는 남아있는다.
- 도커 이미지들은 Dockerhub를 통해 버전 관리 및 배포가 가능하다.
- 다양한 API제공으로 자동화가 가능하다.
- 이미지는 Dockerfile이라는 파일로 이미지를 만든다. 즉, Dockerfile에는 소스 및 의존성 패키지, 설정파일 등 버전관리가 용이하게 명시되어진다. (누구나 이미지 생성을 확인할수있게됨)
  
### 도커 레이어

레이어는 기존 이미지에 추가적인 파일이 필요할때 해당파일을 추가하기 위한것임.
레이어를 사용하면, 다시 다운로드받지않아도된다.
도커 허브에서 해당 이미지의 레이어만을 배포, 다운로드가 가능하다.

### 도커 컨테이너

'도커 이미지'를 실행한 상태를 말한다. 이미지에 담겨있는 프로젝트/응용프로그램의 종속성과 함께 자체를 캡슐화하며 격리된 공간에서 프로세스를 동작시키는 기술을 말한다.
쉽게말하면 도커 이미지를 실행한것.
도커 컨테이너와 도커 레이어는 서로 연관되어있다.

- 컨테이너는 이미지레이어에 읽기/쓰기로 레이러를 추가하는것으로 생성 및 실행이된다. 여러개의 컨테이너를 생성해도, 최소한의 용량이 사용되는것이다.

- 컨테이너는 종료해도 메모리에남아있으므로, 명시적으로 삭제해줘야한다. 다만 명시적으로 삭제하지않으면 메모리에 그대로 남아있기때문에 다시 시작하기 용이하다.


## 도커 이미지 생성을위한 도커 파일(DockerFile)을 만들어보자

이제 도커 이미지, 즉 도커파일을 만들어보자.

그전에, Docker파일을 만들때 Docker파일의 이미지 이름을 Dockerfile이라고 지으면, build할때 별다른 옵션이 필요하지않다. Docker파일의 이름은 이미지화 시키려는 응용프로그램/프로젝트에따라 다르게 지을수있겠지만, 이름을 다르게 지을시 빌드때 다른옵션이 필요하다.


도커 이미지를 빌드하기위해 도커파일내부 옵션을 보자

```python
여러옵션이있지만, 아래옵션을 가장많이쓴다.

FROM
CMD
RUN
WORKDIR

```


### 도커파일을 이미지파일로 빌드

dockerfile을 만들어놓은 경로로 이동해서 작업하길 추천.

```bash
기본)

>docker build 옵션 경로 url  

옵션)

-t : 이미지파일 태그지정

예시)
>docker build -t first-test:ver_1 .


위와같이 명령어를 실행하면,

이미지파일 이름: first-test
태그: ver_1

이렇게 생성된다. 저기서 .는 해당경로안의 dockerfile을 찾아 이미지파일로 만들겠다는 의미이다.
즉, . 자리에는 경로가와야한다. 만약 도커파일의이름이 dockerfile이 아닌경우, 이름을 적어줘야한다.

#bulid시 지정한 이름의 dockerfile을 찾지못할경우 docker와 연결되어있는 계정의 도커허브의 레포지터리에서 pull을 실행해 파일을 받아온다.

```

### 이미지 파일로 빌드하였으면, Docker Hub에 push해보자

빌드한 이미지파일을 허브에도올려서 관리가 쉽도록한다.

```bash
Docker Hub를 이용하기 위해 로그인을 먼저해준다.

>docker login

Docker Hub를 tag로 열어준다.

>docker tag first-test:ver_1 유저이름/이미지이름:태그

유저이름:레포지터리이름
이미지이름:올리려는 빌드된 이미지파일이름
태그:올리려는 빌드된 이미지파일이름의 태그

ex)
만약 허브의 레포지터리이름이 docker_test라면,

>docker tag  docker_test/fist-test:ver_1 

원하는 이미지를 push한다.

>docker push 유저이름/이미지이름:태그

ex)

>docker push docker_test(레포지터리가칭)/fist-test(이미지명):ver_1(이미지의태그명)


이후, 도커허브에 접속하여 잘 올라갔는지 확인한다.
```
![도커 허브확인](/docker_Manual/Docker_hub_1.PNG)

도커에 로그인해서 허브에들어가보면 확인이가능하다.

![도커_데스크톱_로컬확인](/docker_Manual/Docker_image_1.PNG)

**도커 데스크톱**을 이용하여 로컬에있는 이미지파일들, 실행중인 컨테이너들을 볼수있고, 허브도 확인할수있다.
위 사진을 보면 이미지파일과 허브에올린 이미지파일, 같은 도커파일이 이름만다르게 올라간걸 확인할수있다.
도커파일은 같으므로 하나는 삭제해도 무관할것같다.


## Docker를 서버(GCP)에도 설치하자

아까 업로드해놓은 Docker이미지를 서비스를 배포할 서버에 가져와야한다.
여러방법이있겠지만 아까 Docker Hub에 업로드해놓았으니, 서버에도 Docker를 설치하여 Hub를 통해 이미지파일을 다운받아볼것이다.

GCP를 통해 배포할것이므로 GCP콘솔에 접속하여 Docker를 설치한다.
내가 사용하는 GCP의 가상머신이미지는 **Ubuntu-20.04.5**이므로, 우분투 설치메뉴얼을 따른다.

[도커설치_docs_우분투](https://docs.docker.com/engine/install/ubuntu/)

Local Docker버전이 20.10.22버전이므로, 서버에도 동일한 버전의 도커를 설치할것

### 도커설치 코드

```bash

#apt 저장소를 이용하는 방법을 사용


##특정 버전을 설치하기위해 검색 및 설치

### 설치가능한 버전 검색
apt-cache madison docker-ce | awk '{ print $3 }'

### 원하는 버전 선택
VERSION_STRING=5:20.10.22~3-0~ubuntu-focal

###선택한 버전 설치
sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin 

### 도커에서 기본으로 제공하는 테스트 이미지를 실행하여 설치 테스트진행

sudo docker run hello-world
```

[GCP도커설치성공](/docker_Manual/Docker_GCP_install_step1.PNG)

설치에 성공하면 기본제공이미지인 hello-world 이미지가 정상실행된다.
이후

>docker -version / docker create 등 docker 명령어 사용가능

**단, docker 명령어 사용시 앞에 sudo를 붙여줘야한다**
기본적으로 도커데몬이 root로실행되고있기때문이라고한다. 
일반 계정사용자로 docker명령어를 이용하고싶다면 아래 Docker공식문서를 참고

[Docker_docs_사용자로데몬실행](https://docs.docker.com/engine/security/rootless/)
[Docker_docs_사용자로관리하기](https://docs.docker.com/engine/install/linux-postinstall/)

### Docker Compose

Docker compose란?
docker-compose는 복수 개의 컨테이너가 유기적으로 묶여서 하나의 도커 애플리케이션으로 동작할 수 있도록 구성하는 도구이며, 복수 개의 컨테이너 생성 및 실행을 자동화하고 관리하는 기능을제공하는것인데 예전엔 수동으로 설치해야했지만, 최근나오는 Docker패키지에 포함되어있는 기능이다.

기본적으로 Docker-compose.yml파일을 읽어서 해당 파일안에 설정된 컨테이너의 정의에따라 컨테이너를 생성한다.
그러므로, Docker-compose.yml파일을 사용자용도에맞게 작성하여 생성한다.

추가적 옵션이나 사용방법은 아래 링크참조

[Docker Compose_Doc](https://docs.docker.com/compose/)

아래 명령어로 설치되어있는지 버전확인

> docker-compose -v

