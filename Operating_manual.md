# Spark 기본 조작 매뉴얼(우분투)

## Rdd를 하나 생성하여 하둡에 파일을 올려보자

★하둡에서 명렁어 사용시, hdfs '-'명령어 옵션 이다. 앞에 '-'를 붙이면 대부분 O

EX)하둡에 올라가있는 특정 디렉터리삭제 :  hdfs dfs -rm [옵션] 삭제할폴더경로

```python

1.pyspark
test=[1,2,3,"Abc,Def",'Gh'] 테스트 데이터 생성
rdd01=sc.parallelize(test,3) parallelize를 통해 rdd로 변환. 파티션은3개
rdd01.collect() rdd 확인.

rdd01.saveAsTextFile('/rddtest01') 하둡에 올리기 rddtest01이라는 디렉터리안에(없으면 생성)

2.하둡에 파일 저장(올리기)

1.로컬 터미널에서 파일저장
hdfs dfs -put 원본파일명 하둡에서부를파일명

2.pyspark에서 저장

csv write(저장, 포맷은 여러가지이지만 대표적인게 csv)

card2010=spark.read.option('header','true').csv('card_data/csv/202010.csv')

card2010.write.format('csv').mode('overwrite').save('/temp/csv')

↓↓
json을 저장하고싶으면 포맷을 바꿔주면된다.

jsonDf.write.format('json').mode('overwrite').save('/tmp/json')
->다만 일반적인 {kye:value, key:{value,value...}}의 형태가아니라 {} {} {}..형태이다.

3.하둡 디렉터리권한 변경

hdfs dfs -chmod 부여할권한 /경로
EX) hdfs dfs -chmod 444 /test 

```
## 하둡에서 파일을내려받기

```python

1.헤더옵션없이 그냥가져오기

card2010 =spark.read.csv('card_data/csv/202010.csv')
-> 컬럼명이 _c0 _c1...이렇게출력

2.헤더옵션을줘서 컬럼명도 출력되게

card2010=spark.read.format('csv').option('header','true').load('card_data/csv/202010.csv')
-> 컬럼명이 생겨서 이용일, 결제일,금액...으로나옴

3.원하는구조의 데이터프레임에 넣어서 가져오기

EX)한글csv->영어로 가져와보자

from pyspark.sql.types import *

engColumns = StructType([
	StructField('postdate', IntegerType(), True),
	StructField('transdate', IntegerType(), True),
	StructField('amount', IntegerType(), True),
	StructField('pointree', IntegerType(), True),
	StructField('maincategory', StringType(), True),
	StructField('subcategory', StringType(), True),
	StructField('store', StringType(), True)
])

StructType: 구조체유형(구조체x).row이다. StructFiedl와 한묶음이라고 생각
StructField: StructType과 한묶음. ()안에 각  컬럼명,컬럼유형(string,integer..),T/F
add : 위와같이 StructType([StructField(내용),StructField(내용)..]) 한번에 넣을수도있지만, StructType().add(컬럼명1,타입,T/F).add(컬럼명2,타입,T/F)로할수도있다.

engColumns = StructType([
	StructField('postdate', IntegerType(), True),
	StructField('transdate', IntegerType(), True))]
	↓↓↓↓
engColumns = StructType().add('postdate',IntegerType(),True).add('transdate',IntegerType(),True)

두개의 결과는같다.

card2010=spark.read.format('csv').option('header','true').schema(engColumns).load('card_data/csv/202010.csv')
->card2010을 아까 위에서 만든 engColumns틀에 넣는다.

4.json파일을 불러오기

json2010 =spark.read.format('json').load('card_data/json/202010.json')
-> #json2010=spark.read.json('card_data/json/202010.json')

5. mysql에서db파일 불러오기&저장

mysql접속정보 변수선언

user='계정명'
password='비밀번호'
url='jdbc:mysql://localhost:(mysql포트번호)/mysql'
driver='com.mysql.cj.jdbc.Driver'
dbtable='테이블 이름'

5-1.불러오기/여러줄
testDf=spark.read.format("jdbc").option("user",user).option("password",password).option("url",url).option("driver",driver).option("dbtable",dbtable).load()

5-2.불러오기/한줄 option->options
spark.read.format("jdbc").options(user=user, password=password, url=url, driver=driver, dbtable=dbtable).load()

5-3.spark->mysql 저장

저장할Df.write.jdbc(url,dbtable,"append",properties={"driver":driver, "user":user,"password":password})
->옵션이 append이기때문에 변수에 저장되어있는 유저테이블에 저장df내용 추가한다.

저장모드 설정
- append : 추가
- overwrite : 덮어쓰기
- errorifexists : 이미 다른파일이 존재할경우 오류를 발생시킴

```

# bash shell service 관련

※권한이없을경우 sudo를 앞에붙인다는 가정

|이름|설명|
|:---:|:---:|
|systemctl daemon-reload|서비스등록|
|systemctl enable 서비스명|서비스 자동활성화|
|systemctl start 서비스명|서비스 시작|
|systemctl status 서비스명|서비스 상태|
|systemctl stop 서비스명|서비스 중지|
|systemctl disable 서비스명|서비스 비활성화|

## Mysql 유저와 권한설정

```sql

#접속하기

sudo mysql -u 계정명 -p

1-1.sudo mysql -u root
->설치 후 처음접속할때만 가능비밀번호가 초기화되어있는상태라서 비밀번호없이 접속가능

#해당유저 비밀번호바꾸기
alter user '계정명'@'localhost / %' identified with mysql_native_password by 

#유저생성하기
create user '계정명'@'localhost / %' identified by 비밀번호입력

#권한적용하기
flush privileges;

- with grant option : grant명령어 사용할수있는 권한추가
- revoke grant option : 권한해제
- show grants for '계정명'@'localhost / % / ip주소' : 해당계정 권한 조회
- select user()  OR select current_user(); : 현재접속해있는 유저정보 출력
- show databases; : 현재생성되어있는 데이터베이스 목록


```

## CURL 사용 & RESTFUL 사용

★장고에서도 REST사용가능 
[장고REST](https://www.django-rest-framework.org/)

GET:GET방식으로 통신하면 서버에서 데이터를 **받아올것**임
POST:POST 방식으로 통신한다고하면 서버에 데이터를 추가,생성 할것이다.만들어진 데이터까지 번호를할당하고 제일 마지막에만들어진 데이터 뒤에 ID가 할당된다.

# 리눅스 기본명령어

rm ~*.확장자명 :특정 확장자로 끝나는 파일 모두삭제
rm -f 파일명 : 강제삭제

-----


# 크롤링 

|패키지명|설명|
|:---:|:---:|
|셀레니움|동적인것(ajax, js를 쓰는 경우)을 크롤링해오는데 자주사용. 문법이 쉽고 매크로처럼 웹을 브라우저를 조작.</br>속도가 느림|
|뷰티플스푸|정적인것을 크롤링해오는데 자주사용. requset와 같이 흔히 사용하는 라이브러리</br>.속도가 빠름|


## 셀레니움

동적인것이란 무엇일까?
ajax,js쓰는경우가 가장 대표적인 동적페이지이다.

1.그 페이지의 어떤요소를 크롤링해오는데 값이 아무리해도 나오지않음

2.주소가바뀌지않음(비동기)

크롬 / 파이어폭스 /사파리 를 이용할수있다.

각 웹브라우저 버전에맞는 웹드라이버를 설치해줘야하며, 사파리는 기본탑재이므로 설치X
다만 WebdriverMnager가 나와서 크롬이나 파이어폭스도 버전에 자유롭게 사용할수있게되었다.

사람바이사람이지만 문법이 좀 더 간편하다.

|웹브라우저명|설명|
|:---:|:---:|
|크롬| 윈도우에서 사용 추천. 웹드라이버 $PATH를 설정해줘야함|
|파이어폭스|리눅스에서 사용추천. 웹드라이버 $PATH를 설정해줘야하고, 리눅스환경에서 사용 시</BR>디스플레이 문제때문에 파이어폭스 옵션에 --headless를 추가해줘야한다.|
|사파리|위의 두 브라우저와달리 **option값을 따로 지정해주지않아도된다**,**$PATH지정x**</br>단,우클릭으로 개발자용>원격 자동화 허용 활성화 필수!|


- option("--headless") : 웹브라우저를 열지않고 크롤링할때 사용한다.
- wait(시간) : 웹페이지 로딩을 지정시간동안 기다려준다. 지정시간동안 로딩이끝나면 다음코드실행
- click() : .click앞에있는 **특정요소**를 클릭해준다.

셀레니움 사용시 import 목록과 기타 기능

|이름|설명|예시|
|:---:|:---:|:---:|
|By|from셀레니움.웹드라이버.common.by import By|웹드라이버변수.find_elements</BR>(By.Class_name,요소네임)|
|wait|from셀레니움.웹드라이버.support.ui import WebDriverWait|WebDriverWait(dirver, 5)|
|webderiver|from셀레니움 import webdriver|웹드라이버변수=webdriver.Firefox(service=서비스변수,</BR> options=옵션변수)
|
|Service|from셀레니움.webdriver.firefox.service import Service|서비스변수=</BR>Service('geckodriver의$PATH값(경로)')|
|option|from셀레니움.webdriver import FirefoxOptions|옵션변수 = FirefoxOptions()</br>옵션변수.add_argument("--headless")|
|find_element|하나만 선택해서 가져온다|-|
|find_elements|여러개를 선택해서 가져온다.값이없으면 **빈 리스트를 반환**|-|
|웹드라이버변수명.get('주소')|get안에있는 주소에접속|-|

---

**By모듈에대한 설명**

|이름|설명|
|:---:|:---:|
|By.TAG_NAME|태그로 선택한다|
|By.CSS_SELECTOR|CSS선택자를 이용해서 선택한다|DIV.WORDTEST il (DIV클래스명이 WORDTEST이고</BR>그 안의 il태그들을 가져온다) '>' ( < 자식태그하나만가져오는것) 공백으로 구분하면 하위개념 |
|By.LINK_TEXT|지정한 텍스트가있는 태그를 찾아 A태그의 링크를찾아옴|
|By.CLASS_NAME|클래스네임으로 선택한다 .클래스네임|
|By.ID|아이디로 선택한다 #아이디|
|By.NAME|name속성을 이용한다|
|By.PARTIAL_LINK_TEXT|링크텍스트 일부만 일치하는것으로 링크를가져옴|
|By.XPATH|XAPTH를 이용한다|



## 뷰티풀스푸

리퀘스트와 단짝. 기본적으로 

```python
import 리퀘스트
from bs4 import 뷰티풀스푸

url='주소'

resp(변수)=requset.get(url)
soup=뷰티플스푸(resp.text,'html.parser')

위와같이쓰는데,

select
select_one
find
find_all

네가지가있다. 차이점은 셀레니움과같음. 단수/복수 의 차이

```