# Spark 기본 조작 매뉴얼(우분투)

## Rdd를 하나 생성하여 하둡에 파일을 올려보자

★하둡에서 명렁어 사용시, hdfs '-'명령어 옵션 이다. 앞에 '-'를 붙이면 대부분 O

EX)하둡에 올라가있는 특정 개별파일삭제 :  hdfs dfs -rm [옵션] 삭제할폴더경로

보통,하둡은 권한이 설정되어있어서 삭제작업할때 매우귀찮다.
그리고 디렉터리안에 파일이들어있으면 삭제가되지도않기때문에, 삭제방법에 대해 알아보자

    1.디렉터리를 터미널에서 권한변경없이 통째로삭제
    hdfs dfs -rm -r -skipTrash /folder_name

    2.파일을 터미널에서 권한변경없이 삭제
    hdfs dfs -rm -r -skipTrash /file_name 
    hdfs dfs -rm -r /file_name or 경로

    3.웹브라우저로들어가 삭제하는법
    hdfs dfs -chmod 773(<원하는권한) /folder_name or file 경로

## SPARK DF <-> PANDAS DF/ DF에 ID를자동생성하고 없애기

```PYTHON
#형태변환
result_kbs = kbs_idx.select("*").toPandas()
result_sbs = sbs_idx.select("*").toPandas()
result_kukmin = kukmin_idx.select("*").toPandas()
result_daily = daily_idx.select("*").toPandas()

concatDF=pd.concat([result_kbs,result_sbs,result_kukmin,result_daily])
news_concatdf = spark.createDataFrame(concatDF)

#ID재부여

news_concat_id=news_concatdf.rdd.zipWithIndex().toDF()
final_newsDF=news_concat_id.select(col("_2").alias("id"),col("_1.*"))
->COL("_1.*"),COL("_2")를쓰면 아이디가 맨뒤에가서붙는다.
```

## 정규식을 사용하여 원하는걸 삭제해보자
```PYTHON
아래의 모듈 import필수!
from pyspark.sql import *
from pyspark.sql.functions import regexp_replace
from pyspark.sql.functions import col
import re

#main컬럼 전처리. 아래와같이 []를사용하여 여러개를 한번에 처리한다.
main_p=sbs_news.select(col("title"),regexp_replace(col("main"),'[\n▶▲■▷▼●\[\]]'," ").alias("main"),col("date"),col("url"))
# title 컬럼에대하여 정규식을 사용해 여러문자 제거. []는 특수한문자이므로 이스케이프가 필요하여 앞에 \를 사용. []안에있는 문자들과 일치하면 모두 공백으로변경
title_p=main_p.select(regexp_replace(col("title"),'[\[,"\]+]',"").alias("title"),col("main"),col("date"),col("url"))

#<>나 (), []안의 문자제거
title_p=main_p.select(regexp_replace(col("title"),'[,]',"").alias("title"),col("main"),regexp_replace(col("date"),'\([^)]*\)',"").alias("date"),col("url"))

중요 : '\([^)]*\)' ()안에있는 모든 문자를 삭제하겠다는의미이다. () -> <>바꿔지면 <>안에있는 문자 모두삭제.

```



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
- sleep(시간) :from time import sleep. wait과 import해오는 모듈자체도 다를뿐더러 sleep은 **프로세스자체를 지연**시킨다. (wait은 다음코드 실행까지의 지연시간)

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

**Wait 상세**

Implicitly Wait (암묵적 대기)

- Selenium에서만 사용되는 특수한 메서드
- 지정한 시간만큼 대기 (브라우저에서 사용되는 엔진 자체에서 파싱되는 시간을 기다림)
- 모든 요소에 적용
- sleep보다 낭비되는 시간이 적음
- driver.implicitly_wait(10) < 별도의 import 작업 x

Explicitly Wait (명시적 대기)

- Selenium에서만 사용되는 특수한 메서드
- 조건이 True가 될 때 까지 지정한 시간만큼 대기
- 특정 요소에 적용
- from selenium.webdriver.support import expected_conditions as EC

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

## 셀레니움으로 웹페이지 스크롤 업/다운 조작

웹드라이버변수.**execute_script()** 을 이용하자!

execute_script(js작성)메서드는 ()안의 js가 어떤이벤트를 작성하느냐에따라

여러 동적이벤트를 발생시킬수있다. 화면을 조작하는 js를 작성해보자

- 화면상 스크롤 위치 이동 : scrollTo(x,Y) ,scrollTo(x,Y+number)
- 화면 최하단으로 스크롤 이동 : scrollTo(0, document.body.scrollHeight)
- 화면을 움직이고 페이지 로드 기다리기 : time.sleep(second)

```python
예시

SCROLL_PAUASE_TIME=1.5
browser.get('자동스크롤하려는 주소') 
while True:
    sleep(SCROLL_PAUASE_TIME) ->  그냥 sleep(1.5)도 가능
    #스크롤을 내려준다
    last_height = browser.execute_script("return document.body.scrollHeight") -> 끝인지 더내릴곳이있는지 판단하기위해 
    browser.execute_script("window.scrollTo(0, document.body.scrollHeight);")
    sleep(SCROLL_PAUASE_TIME)
    new_height = browser.execute_script("return document.body.scrollHeight")
    if new_height == last_height:
        browser.execute_script(
            "window.scrollTo(0, document.body.scrollHeight);")
        sleep(SCROLL_PAUASE_TIME)
        new_height = browser.execute_script(			->끝인지 더 내릴곳이있는지 판단하기위해. last와 new 높이를 비교한다. 
            "return document.body.scrollHeight")
        if new_height == last_height:		->그리고 같으면 더 내릴곳이없으니까 while문을 멈춘다. 그 전까지는 무한반복
            break
        else:
            last_height = new_height
            continue

출처:https://softmagic.tistory.com/120

```

## 원하는 포맷으로 CSV저장

```python

output = 'test.csv' #저장할 파일명 | 최종 아웃풋 파일명
csv_open = open(output, 'w+', encoding='utf-8') #불러온파일에 덮어쓰기한다.
csv_writer = csv.writer(csv_open)#불러온파일에 덮어쓰기한다
csv_writer.writerow(('키1','키2','키3', '키4')) # 컬럼같은 개념. 타이틀row의 인자값


    키1     = li.find('span', {'class':'요소클래스명'}).text 
    키2     = li.find('span', {'class':'요소클래스명'}).text
    키3    = li.find('span', {'class':'요소클래스명'}).text
    키4 = check_image_url() #이미지 uri저장부분

# CSV 에 저장하자
csv_writer.writerow((키1, 키2, 키3, 키4))

```

## CSS선택자-속성선택자

TAG/CLASS/ID/속성 선택자중 속성선택자를 알아보자

```PYTHON
#div클래스네임이 abc이고 그안의 a태그의 href(링크)주소를 가져와보자

title=browser.find_element(By.CSS_SELECTOR,'abc a')
print(title.get_attribute('href'))
#링크주소를 가져오는데 왜 by.link_text 을 안쓰냐면 text내용이없다는 가정하에 링크를가져온다.link_text는 text내용이있어야 가져올수있으니까.

※중요 .get_attribute() 속성은 elements에 적용되지않는다. 왜냐면, 한개당 속성이 여러개가아니고 하나이기때문에 찾는요소1개=찾는요소속성 1 개 이렇게 생각해야한다.
고로, 단일로찾을거면 위와같이, 여러개를 찾을거면 아래와같이 작성해야한다.

#elements로 링크 여러개를가져오자
title = browser.find_elements(By.CLASS_NAME, 'abc a')

for element in title:
    print(element.get_attribute('href'))

★for문으로 돌려주면 단일 한줄, 한줄씩 실행되는거니까 여러개의 링크를 가져올수있다.

#title이 가져오는 총 개수를알고싶을때
 print(len(title)) 


#title의 text부분(text == html_text == inner_text)만 가져오고싶을때
for comment in title:
	print(comment.text)

```
## BeautifulSoup

리퀘스트와 단짝. 기본적으로 

```python
import requests
from bs4 import BeautifulSoup

url='주소'

resp(변수)=requests.get(url)
soup=BeautifulSoup(resp.text,'html.parser')

위와같이쓰는데,

select
select_one
find
find_all

네가지가있다. 차이점은 셀레니움과같음. 단수/복수 의 차이

```

# Python

## 리스트 중복제거하기

- functools.reduce() : 리스트의 기존순서를 유지하면서 중복을 제거한다. for문처럼 리스트를 돌면서 현재값이 리스트안에 없는값이면 리스트에 추가시키는 방식으로 동작한다. 별도의 import 필요

        from functools import reduce

        array = ["F", "D", "A", "C", "A", "C", "F", "B", "C", "E", "D", "C", "F", "A", "B", "E", "F", "E"]

        result = reduce(lambda acc, cur: acc if cur in acc else acc+[cur], array, [])

-  dict.fromkeys(): 기존 순서를 유지하면서 중복을제거한다. (파이썬 3.7부터 dict가 순서를 보존함)

            
        result = list(dict.fromkeys(array))
    
  
- for문 :기존순서를 유지하면서 중복을제거한다.리듀스와 같은방식으로 중복제거하는데 **인덱스의 순서대로 for문을 순환** 하여 기존순서가 유지됨
 
        result = []
        for value in array:
            if value not in result:
                result.append(value)

- set() : **기존순서를 유지하지 않고**중복을 제거한다.중복을 허용하지않고 순서가없는 특성을 이용해 중복제거를한것. 

        result = list(set(array))


## ELK

### elesticsearch

- 인덱스 : 테이블
- 도큐먼트(doc) : 데이터
- 매핑 : 스키마
- 컬럼 : 필드
  
고로, 도큐먼트는 하나의 인덱스에 꼭 포함되어야하고, 인덱스는 매핑작업이 반드시필요하다. 매핑을 생성해주지않으면 입력되는 도큐먼트타입에따라 자동으로 매핑이 생성된다.  날짜데이터나 정확한값이 입력되어야하는경우 인덱스를 생성할떄 매핑작업을 같이해줘야한다.

### logstash

    로그스테시 사용시 기억할점

    >파이프라인을 반드시 설정해야한다. 따로 파이프라인파일을 생성해주지않고 쓰려면, 콘솔창에서 -e옵션을쓸수있다.
    >파이프라인은 기본적으로 입력 > 필터 > 출력 으로 이루어져있고 입력/출력은 필수. 필터는 옵션이다.
    >ex: C:\logstash-7.10.1>.\bin\logstash.bat -e "input { stdin { } } output { stdout { } }"
    >위와같이 -e옵션을 이용하여 cmd창에 입력하고 input이니, 아무거나 입력하면 "message"에 내가입력한 내용이 나오게된다.


## GCP 인스턴스에있는 파일을 원격으로 로컬컴퓨터에 다운로드받기

SCP를 이용하여 GCP인스턴스에있는 파일을 원격으로 로컬컴퓨터에 다운로드받을것이다.

```bash

#로컬 wsl에서 명령어를 실행한다는 기준
#로컬 -> 원격지
> scp [옵션][파일명][원격지id]@[원격지IP]:[파일을보낼경로]

> scp 파일명 gcp계정명@gcp외부ip:파일을보낼경로

#원격지 -> 로컬

> scp [옵션] [원격지_id]@[원격지_ip]:[원본 위치] [받는 위치]

> spc  gcp계정명@gcp외부ip:gcp에서 가져올파일위치 로컬에 저장할 위치

#로컬 -> 원격지 대용량폴더및파일
> tar -cp [복사할 디렉토리 상대경로] | ssh [목적지 주소] tar xvp -C [목적지 디렉토리 절대경로]

```
## GCP에 올려놓은 프로젝트를 실행시켜 외부에서 접속해보자

test instance에 Django 프로젝트를 올려놓았으니 잘 실행되는지 확인해보고 외부에서 접속할수있도록 설정해보자.
또한 추가적으로 보완해야할 부분도 추가해준다.

