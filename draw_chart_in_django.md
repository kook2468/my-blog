# 장고에서 차트 그리는 법

- [How to Integrate Highcharts.js with Django by Vitor Freitas](https://simpleisbetterthancomplex.com/tutorial/2018/04/03/how-to-integrate-highcharts-js-with-django.html)
- [장고 차트 사이트](http://logistex2020.pythonanywhere.com)
- 필요성
    - 쥬피터 랩/노트북 작업 결과물의 배포/공유 어려움
    - 웹 애플리케이션 개발에서 데이터 시각화 도입의 중요성
- 차이점
    - 쥬피터 랩/노트북에서는 csv 파일 및 데이터프레임으로 작업
    - 웹 애플리케이션에서는 DB로 작업
- 핵심 과제
    - 백엔드 서버 단에서 공급되는 데이터를 프론트엔드 클라이언트 단에서 처리되는 'Highcharts'가 이해하는 형식으로 전환하여 전달하는 방법
    - 백엔드 데이터는 DB 또는 외부 API에서 제공되며, (QuerySet, 사전 혹은 리스트와 같은) 파이썬 객체 형태
- 두 방법
    - request/response 주기를 통하여, 템플릿 내부에서 파이썬 또는 장고 템플릿 엔진을 통하여 백엔드 데이터를 프론트 단으로 전달
    - AJAX를 이용하는 비동기 request를 통하여, 백엔드 데이터를 JSON 형식으로 반환하여 프론트 단으로 전달

## 1. Highcharts 맛보기
- 장고와 하이차트의 통신 방법을 익히는 것이 핵심
- 하이차트 [공식 문서](https://www.highcharts.com/docs)
- 아래 코드를 'world_population.html' 파일로 저장한 후 열어보면 차트가 브라우징 됨
```HTML {.line-numbers}
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Highcharts Example</title>
</head>
<body>
  <div id="container"></div>
  <script src="https://code.highcharts.com/highcharts.src.js"></script>
  <script>
    Highcharts.chart('container', {
        chart: {
            type: 'column'
        },
        title: {
            text: 'Historic World Population by Region'
        },
        xAxis: {
            categories: ['Africa', 'America', 'Asia', 'Europe', 'Oceania']
        },
        series: [{
            name: 'Year 1800',
            data: [107, 31, 635, 203, 2]
        }, {
            name: 'Year 1900',
            data: [133, 156, 947, 408, 6]
        }, {
            name: 'Year 2012',
            data: [1052, 954, 4250, 740, 38]
        }]
    });
  </script>
</body>
</html>
```
![](https://simpleisbetterthancomplex.com/media/2018/04/highcharts-column-chart.png)
- 핵심 구조
```
Highcharts.chart('id_of_the_container', {
  // dictionary of options/configuration
});
```
- 몇 가지 사소한 시도
  - div#container 영역에 차트가 작성되는데, div 태그의 id를 'my-container'로 수정 시도
  - chart.type: 'column'(수직 막대), 'bar'(수평 막대), 'line'(꺽은선), 'area'(영역) 시도
  - title.text: '지역에 따른 세계 인구 변동'으로 수정 시도
  - 하이차트 [견본](https://www.highcharts.com/demo)을 참고해서 y축 레이블을 '인구 수 [단위: 백만 명]'으로 수정 시도
- 이런 차트를 장고 웹 앱에서 보여주려면, ...
  - 위와 같은 자료를 DB 등에서 실시간으로 가져와야 한다면, ...
  - 벡엔드 DB에서 자료를 집계하여 템플릿을 통해 프론트 단으로 전달해야, ...

## 2. 장고에서 Highcharts 맛보기
### 2.1 장고 준비 작업

<b>2.1.1 장고 가상환경 확인</b>
```shell {.line-numbers}
$ conda info --envs             # 콘다 가상환경 정보 확인
$ conda activate vnv_vd
$ conda install django          # 장고 설치
$ python -m django --version    # 장고 버전 확인 2.2.5
```

<b>2.1.2 Highcharts 설치</b>
Highcharts 라이브러리를 우리 템플릿에 포함시키는 작업
- 직접 [다운로드](https://www.highcharts.com/download)해서 포함시키는 방법
- 간편하게 CDN을 활용하는 방법
   ```HTML
   <script src="https://code.highcharts.com/highcharts.src.js"></script>
   ```

<b>2.1.3 파이참에서 프로젝트 생성</b>
- 파이참에서 열려 있는 프로젝트를 닫기
- `Create New Project` 클릭하기
- 가상환경을 'vnv_vd'로 지정한 'C:\work' 프로젝트 생성
  - `New Project` 창의 `Pure Python` 메뉴에서
  - `Location`에 'C:\work' 입력하고
  - `Project Interpreter` 클릭하여,
    `Existing interpreter` 선택하고, `...` 단추 클릭하여,
    `Add Python Interpreter` 창에서 `Conda Environment` 클릭하고,
    `Interpreter` 항목의 `...` 단추 클릭하여,
    'C:\Anaconda3\envs\vnv_vd\python.exe' 선택하고,
    `Make available to all project` 체크하고,
    `OK` 단추 클릭한 후,
    `Existing interpreter` 항목의 `Interpreter` 리스트 단추를 확장하면 보이는
    'Python 3.8 (vnv_vd)' 항목을 선택하고
  - `Create` 단추 클릭하여 프로젝트 생성 완료
- 파이참에서 생성된 프로젝트 폴더가 열리면,
  터미널 창을 열어서 프롬프트가 '(vnv_vd) C:\work>'임을 확인하고 작업
  ```shell
  (vnv_vd) C:\work>python -m django --version          # 2.2.5
  (vnv_vd) C:\work>django-admin startproject config .  # 장고 프로젝트 생성
  (vnv_vd) C:\work>python manage.py migrate            # DB 초기화
  (vnv_vd) C:\work>python manage.py startapp chart     # 장고 앱 생성
  ```
### 2.2 장고 코딩 작업

<b>2.2.1 설정 파일 수정</b>
- settings.INSTALLED_APPS 항목 끝에 `'chart', ` 추가
- settings.TEMPLATES[0]['DIRS'] 값에 `os.path.join(BASE_DIR, 'templates'),` 추가

<b>2.2.2 접속 경로 등록</b>
```PYTHON {.line-numbers}
# hChart.urls.py
from django.contrib import admin
from django.urls import path
from chart import views                                     # !!!

urlpatterns = [
    path('world-population/',
         views.world_population, name='world_population'),  # !!!
    path('admin/', admin.site.urls),
]
```

<b>2.2.3 뷰 작성</b>
```PYTHON {.line-numbers}
# chart/views.py
from django.shortcuts import render

def world_population(request):
    return render(request, 'world_population.html')
```

<b>2.2.4 템플릿 작성</b>
- 앞서 저장했던 'world_population.html' 파일을
- 'C:\work\templates' 폴더 속으로 복사

<b>2.2.5 테스트</b>
- 개발 서버 기동 후, 브라우저에서  http://127.0.0.1:8000/world-population/ 접속
- 하이차트 맛보기에서 시도했던 차트 수정 다시 시도해 보기

## 3. 타이타닉 데이터셋 활용

### 3.1 데이터셋 소개
- 타이타닉 비극 [RMS(Royal Mail Ship) Titanic tragedy](https://en.wikipedia.org/wiki/RMS_Titanic)
- 탑승객 1309명에 관한 데이터
```SQL {.line-numbers}
SELECT count() FROM chart_passenger;
```
<table border="1" style="border-collapse:collapse">
<tr><th>COUNT()</th></tr>
<tr><td>1309</td></tr>
</table>
- 첫 5 행

```SQL {.line-numbers}
SELECT * FROM chart_passenger LIMIT 5;
```
<table border="1" style="border-collapse:collapse">
  <tr><th>id</th><th>name</th><th>sex</th><th>survived</th><th>age</th><th>ticket_class</th><th>embarked</th></tr>
  <tr><td>1</td><td>Allen, Miss. Elisabeth Walton</td><td>F</td><td>1</td><td>29</td><td>1</td><td>S</td></tr>
  <tr><td>2</td><td>Allison, Master. Hudson Trevor</td><td>M</td><td>1</td><td>0.9167</td><td>1</td><td>S</td></tr>
  <tr><td>3</td><td>Allison, Miss. Helen Loraine</td><td>F</td><td>0</td><td>2</td><td>1</td><td>S</td></tr>
  <tr><td>4</td><td>Allison, Mr. Hudson Joshua Creighton</td><td>M</td><td>0</td><td>30</td><td>1</td><td>S</td></tr>
  <tr><td>5</td><td>Allison, Mrs. Hudson J C (Bessie Waldo Daniels)</td><td>F</td><td>0</td><td>25</td><td>1</td><td>S</td></tr>
</table>
- 데이터를 `chart`앱의 `Passenger` 모델로 적재할 예정

```python 
class Passenger(models.Model):                                    # 승객
    name = models.CharField(max_length=100, blank=True)             # 이름
    sex = models.CharField(max_length=1, choices=SEX_CHOICES)       # 성별
    survived = models.BooleanField()                                # 생존_여부
    age = models.FloatField(null=True)                              # 연령
    ticket_class = models.PositiveSmallIntegerField()               # 티켓_등급
    embarked = models.CharField(max_length=1, choices=PORT_CHOICES) # 승선_항구
```
### 3.2 작성할 차트 소개
다양한 집계 정보를 조회하여, 차트 작성에 활용해야 함

<b>3.2.1 승선_항구별 인원 집계</b>
![](https://github.com/logistex/vd/blob/master/highchartsWithDjango/images/chart2.png?raw=true)
```Python
[
  {'embarked': 'C', 'total': 270},
  {'embarked': 'Q', 'total': 123},
  {'embarked': 'S', 'total': 914}
]
```
```SQL {.line-numbers}
SELECT embarked, count(embarked)
FROM chart_passenger
GROUP BY embarked;
```
<table border="1" style="border-collapse:collapse">
  <tr><th>embarked</th><th>count(embarked)</th></tr>
  <tr><td></td><td>2</td></tr>
  <tr><td>C</td><td>270</td></tr>
  <tr><td>Q</td><td>123</td></tr>
  <tr><td>S</td><td>914</td></tr>
</table>

<b>3.2.2 티켓_등급별 생존/비생존 인원 집계</b>
![](https://github.com/logistex/vd/blob/master/highchartsWithDjango/images/chart1.png?raw=true)
```Python
[
  {'ticket_class': 1, 'survived_count': 200, 'not_survived_count': 123},
  {'ticket_class': 2, 'survived_count': 119, 'not_survived_count': 158},
  {'ticket_class': 3, 'survived_count': 181, 'not_survived_count': 528}
]
```
```SQL {.line-numbers}
SELECT ticket_class, survived, count()
FROM chart_passenger
GROUP BY ticket_class, survived;
```
<table border="1" style="border-collapse:collapse">
<tr><th>ticket_class</th><th>survived</th><th>count()</th></tr>
<tr><td>1</td><td>0</td><td>123</td></tr>
<tr><td>1</td><td>1</td><td>200</td></tr>
<tr><td>2</td><td>0</td><td>158</td></tr>
<tr><td>2</td><td>1</td><td>119</td></tr>
<tr><td>3</td><td>0</td><td>528</td></tr>
<tr><td>3</td><td>1</td><td>181</td></tr>
<tr><td colspan=3> ... </td></tr>
</table>

### 3.3 csv 파일을 DB로 적재하는 방법

<b>3.3.1 앱 모델 코드 작성</b>
```PYTHON {.line-numbers}
# chart/modles.py
from django.db import models

class Passenger(models.Model):  # 승객 모델
    # 성별 상수 정의
    MALE = 'M'
    FEMALE = 'F'
    SEX_CHOICES = (
        (MALE, 'male'),
        (FEMALE, 'female')
    )

    # 승선_항구 상수 정의
    CHERBOURG = 'C'
    QUEENSTOWN = 'Q'
    SOUTHAMPTON = 'S'
    PORT_CHOICES = (
        (CHERBOURG, 'Cherbourg'),
        (QUEENSTOWN, 'Queenstown'),
        (SOUTHAMPTON, 'Southampton'),
    )

    name = models.CharField(max_length=100, blank=True)                 # 이름
    sex = models.CharField(max_length=1, choices=SEX_CHOICES)           # 성별
    survived = models.BooleanField()                                    # 생존_여부
    age = models.FloatField(null=True)                                  # 연령
    ticket_class = models.PositiveSmallIntegerField()                   # 티켓_등급
    embarked = models.CharField(max_length=1, choices=PORT_CHOICES)     # 승선_항구

    def __str__(self):
        return self.name
```

<b>3.3.2 DB 현행화 준비 실행</b>
- DB 현행화 준비 실행 전에는, `migration` 폴더가 빈 상태(단지 `__init__.py` 파일만 존재)
```shell
(vnv_vd) C:\work>python manage.py makemigrations            # DB 현행화 준비
```
- DB 현행화 준비 실행 후에는, `migration` 폴더에 `0001_initial.py` 파일이 생성됨

<b>3.3.3 csv 파일을 DB로 적재하는 코드 작성</b>
```PYTHON {.line-numbers}
# chart/migrations/0002_auto_popuate.py
"""
DB 현행화 작업이 실행될 때, csv 파일 자료를 DB에 자동적으로 적재한다.
"""
import csv
import os
from django.db import migrations
from django.conf import settings

# csv 파일의 해당 열 번호를 상수로 정의
TICKET_CLASS = 0  # 승차권 등급
SURVIVED = 1      # 생존 여부
NAME = 2          # 이름
SEX = 3           # 성별
AGE = 4           # 나이
EMBARKED = 10     # 탑승지

def add_passengers(apps, schema_editor):
    Passenger = apps.get_model('chart', 'Passenger')  # (app_label, model_name)
    csv_file = os.path.join(settings.BASE_DIR, 'titanic.csv')
    with open(csv_file) as dataset:                   # 파일 객체 dataset
        reader = csv.reader(dataset)                    # 파일 객체 dataset에 대한 판독기 획득
        next(reader)  # ignore first row (headers)      # __next__() 호출 때마다 한 라인 판독
        for entry in reader:                            # 판독기에 대하여 반복 처리
            Passenger.objects.create(                       # DB 행 생성
                name=entry[NAME],
                sex='M' if entry[SEX] == 'male' else 'F',
                survived=bool(int(entry[SURVIVED])),        # int()로 변환하고, 다시 bool()로 변환
                age=float(entry[AGE]) if entry[AGE] else 0.0,
                ticket_class=int(entry[TICKET_CLASS]),      # int()로 변환
                embarked=entry[EMBARKED],
            )

class Migration(migrations.Migration):
    dependencies = [                            # 선행 관계
        ('chart', '0001_initial'),         # app_label, preceding migration file
    ]
    operations = [                              # 작업
        migrations.RunPython(add_passengers),   # add_passengers 함수를 호출
    ]
```

<b>3.3.4 csv 파일을 `BASE_DIR` 폴더에 복사</b>
- [이곳](https://github.com/sibtc/django-highcharts-example.git)에서
  `Clone or download` 단추를 누르고, `download ZIP` 링크를 눌러서
  전체 파일을 다운로드
- 다운로드 받은 ZIP 파일에서 `titanic.csv` 파일을 복사하여
- `BASE_DIR`에 붙여넣기

<b>3.3.5 DB 현행화 작업 실행</b>
```shell
(vnv_vd) C:\work>python manage.py migrate    # DB 현행화
```
- DB 현행화 작업이 성공적으로 수행되면, 자동적으로 DB에 csv 파일 데이터가 적재됨
- 파이참 `Database` 패널에서 적재된 데이터 확인
  - 파이참 `Project` 패널에서 `db.sqlite3` 항목을 더블클릭
  - `Database` 패널에서 `db/schemas/main/chart_passenger` 항목을 더블클릭
  - DB 테이블에 적재된 데이터 확인

## 4. 장고 요약 통계 연습
### 4.1 장고 ORM 기본

<b>4.1.1 장고 쉘 활용 기초</b>
- 파이참 터미널 창에서 `python manage.py shell` 입력하여 쉘 실행
- 먼저 앱(passengers)의 모든 모델을 임포트해야 함
```Python {.line-numbers}
# 모델 임포트
from chart.models import *   # 앱_이름.models
```
- 장고에서 일반적인 검색 결과는 객체 형태 리스트인 QuerySet 객체
- 객체를 출력할 때 모델 클래스에서 정의한 `__str__()` 함수에 의존함
```Python {.line-numbers}
## 특정 모델의 모든 행 검색
Passenger.objects.all()           # 모델_이름.objects.all()
#  <QuerySet [
#    <Passenger: Allen, Miss. Elisabeth Walton>,
#    <Passenger: Allison, Master. Hudson Trevor>,
#    <Passenger: Allison, Miss. Helen Loraine>,
#     '...(remaining elements truncated)...'
#  ]>
```
- 특정 모델의 특정 조건 행 검색
  - get(): 검색 결과가 단일 객체일 때 사용
  - filter(): 검색 결과가 다수 객체일 때 사용
```Python {.line-numbers}
Passenger.objects.get(id=1)             # 단일 행 검색은 get()
#  <Passenger: Allen, Miss. Elisabeth Walton>
```
```Python {.line-numbers}
Passenger.objects.filter(age__gte=75)   # 다수 행 검색은 filter()
#  <QuerySet [
#    <Passenger: Barkworth, Mr. Algernon Henry Wilson>,
#    <Passenger: Cavendish, Mrs. Tyrell William (Julia Florence Siegel)>
#  ]>
```

<b>4.1.2 검색 연산자</b>
```Python {.line-numbers}
# 크기 비교
Passenger.objects.filter(age=80).count()                      #    1
Passenger.objects.filter(age__gt=80).count()                  #    0
Passenger.objects.filter(age__gte=80).count()                 #    1
Passenger.objects.filter(age__lt=0).count()                   #    0
Passenger.objects.filter(age__lte=0).count()                  #  263
Passenger.objects.filter(age__isnull=True).count()            #    0
# AND / OR 결합
Passenger.objects.filter(age__gte=70, age__lt=80).count()     #    7 (AND 결합)
(Passenger.objects.filter(age__lt=70) 
| Passenger.objects.filter(age__gte=80)).count()              # 1302 (OR 결합)

# 문자열 비교
Passenger.objects.filter(name__startswith='Blank').count()    #    1
Passenger.objects.exclude(name__startswith='Blank').count()   # 1308
Passenger.objects.filter(name__endswith='Scott').count()      #    1
Passenger.objects.filter(name__contains='Scott').count()      #    3

# 날짜/시간 비교 (birthday 열이 존재한다고 가정하고)
Passenger.objects.filter(birthday__date=datetime.date(2010, 12, 1))
Passenger.objects.filter(birthday__date__gt=datetime.date(2010, 12, 1))
Passenger.objects.filter(birthday__year__lte=2010)
Passenger.objects.filter(birthday__month=12)
Passenger.objects.filter(birthday__day__gt=30)
Passenger.objects.filter(birthday__week_day=1)  # 일요일
Passenger.objects.filter(birthday__time=datetime.time(14, 30))
Passenger.objects.filter(
  birthday__time__range=(datetime.time(8), datetime.time(17))
Passenger.objects.filter(birthday__hour__gte=23)
```

<b>4.1.3 Q 객체</b>
- <u>filter()</u> 내부에 다수 조건을 ','로 구분하면 묵시적 AND 조건으로 처리됨

- SQL에서 <u>WHERE 절과 유사</u>한 역할
```Python {.line-numbers}
Passenger.objects.filter(
  age__gte=75, sex='M'                  # 조건을 ','로 결합 (묵시적 AND)
)
#  <QuerySet [<Passenger: Barkworth, Mr. Algernon Henry Wilson>]>
```
- filter()로는 OR 검색을 처리할 수 없고, Q 객체를 써야 함
- Q 객체 임포트
```Python {.line-numbers}
from django.db.models import Q          # 우선 Q 객체 임포트
```
- Q 객체로 OR 검색
```Python {.line-numbers}
Passenger.objects.filter(
  Q(age__gte=75) | Q(age__lte=0)        # 조건을 '|'로 결합 (명시적 OR)
)
#  <QuerySet [
#    <Passenger: Allison, Master. Hudson Trevor>,
#    <Passenger: Barkworth, Mr. Algernon Henry Wilson>,
#    '...(remaining elements truncated)...'  # 연령 미상이면 모두 0으로 입력?
#  ]>
```
- Q 객체로 AND 검색
```Python {.line-numbers}
Passenger.objects.filter(
  Q(age__gte=75) & Q(sex='M')           # 조건을 '&'로 결합 (명시적 AND)
)
#  <QuerySet [<Passenger: Barkworth, Mr. Algernon Henry Wilson>]>
```

<b>4.1.4 values()</b>
- 검색 결과에서, `__str__()`이 지원하지 않는, 다른 열 값을 직접 확인하려면
  <u>values()</u> 함수를 활용
  - values() 함수에 열 이름을 나열하면,
    해당 열 이름을 키로 지정하여 값을 함께 보여주는
    <u>사전(dictionary)</u> 형태로 결과를 확인 가능함
  - SQL에서 <u>SELECT 절과 유사</u>한 역할
```Python {.line-numbers}
Passenger.objects.filter(age=80)  # 이름만 확인 가능
# <QuerySet [<Passenger: Barkworth, Mr. Algernon Henry Wilson>]>
# 객체 리스트 반환
Passenger.objects.filter(age=80) \
  .values('age', 'name')          # 나이와 이름 확인 가능
# <QuerySet [{'age': 80.0, 'name': 'Barkworth, Mr. Algernon Henry Wilson'}]>
# 사전 리스트 반환
```
- values(): 검색 결과를 (모델 객체의 리스트가 아닌) 사전 객체의 리스트로 반환
```Python {.line-numbers}
Passenger.objects.filter(age__gte=80)          # 모델 객체의 리스트
#  <QuerySet [<Passenger: Barkworth, Mr. Algernon Henry Wilson>]>
Passenger.objects.filter(age__gte=80).values() # 사전 객체의 리스트
#  <QuerySet [{'id': 15, 'name': 'Barkworth, Mr. Algernon Henry Wilson', 'sex': 'M', 'survived': True, 'age': 80.0, 'ticket_class': 1, 'embarked': 'S'}]>
#]>

Passenger.objects.all()
#  <QuerySet [
#    <Passenger: Allen, Miss. Elisabeth Walton>,
#    <Passenger: Allison, Master. Hudson Trevor>,
#    '...(remaining elements truncated)...'
#  ]>

## SELECT age FROM ...
Passenger.objects.values('age')
#  <QuerySet [
#    {'age': 29.0},
#    {'age': 0.9167},
#    '...(remaining elements truncated)...'
#  ]>

## SELECT age, name FROM ...
Passenger.objects.values('age', 'name')
#  <QuerySet [
#    {'age': 29.0, 'name': 'Allen, Miss. Elisabeth Walton'},
#    {'age': 0.9167, 'name': 'Allison, Master. Hudson Trevor'},
#    '...(remaining elements truncated)...'
#  ]>

## 1세 승객의 연령과 이름만 검색 (아래 두 코드는 같은 결과)
Passenger.objects.values('age', 'name').filter(age=1)
Passenger.objects.filter(age=1).values('age', 'name')
#  <QuerySet [
#    {'age': 1.0, 'name': 'Becker, Master. Richard F'},
#    {'age': 1.0, 'name': 'Laroche, Miss. Louise'},
#    ...
#    {'age': 1.0, 'name': 'Sandstrom, Miss. Beatrice Irene'}
#  ]>
# values()는 SELECT 절과 유사하며,
# filter()는 WHERE 절과 유사함
```

<b>4.1.5 count()</b>
- count(): 일반적 개수를 스칼라 값으로 반환
```Python {.line-numbers}
# 전체 승객 수
Passenger.objects.count()
#  1309

# 0세 승객 수
Passenger.objects \
  .filter(age=0) \
  .count()   # 조건 검색 행 수
#  263 (연령 미상인 경우 0으로 입력한 듯)

# 0세 초과이고 1세 미만
Passenger.objects \
  .filter(Q(age__gt=0) & Q(age__lt=1)) \
  .count()
#   12

# 80세 이상
Passenger.objects \
  .filter(Q(age__gte=80)) \
  .count()
#   1

## (0세 초과 그리고 1세 미만) 또는 (80세 이상)
Passenger.objects \
  .filter((Q(age__gt=0) & Q(age__lt=1)) | Q(age__gte=80)) \
  .count()
#  13
```

### 4.2 장고 ORM 통계
- 집계의 두 가지 방법
  - <u>aggregate</u>: 쿼리셋 전체를 대상으로 집계
  - <u>annotate</u>: 쿼리셋 내부 항목별로 집계 (<u>GROUP BY</u> 집계)
- aggregate: 쿼리셋 전체를 대상으로 집계
  - 승객 전체를 대상으로 인원 수
  - 승객 전체를 대상으로 평균 연령
  - 승객 전체를 대상으로 최대 연령
- annotate: 쿼리셋 내부 항목별로 집계 (GROUP BY 집계)
  - 승선_항구에 따른 승객 인원
  - 티켓_등급에 따른 승객 인원
  - 티켓_등급에 따른 생존 인원
  - 셩별 구분에 따른 생존 인원
  - 연령_구간(?)에 따른 생존 인원

<b>4.2.1 aggregate() 함수로 전체 집계</b>
- count()와 달리, aggregate()는 결과를 사전 형식으로 반환
- aggregate() 함수 인자로 Count(), Min(), Max(), Sum(), Avg() 등을 지정
- aggregate() 함수 인자를 지정할 때 Count('열_이름') 형태로 지정
```Python {.line-numbers}
# 먼저 집계 함수 임포트
from django.db.models import Count, Min, Max, Sum, Avg
```
```Python {.line-numbers}
# count()는 결과를 단일 값으로 반환
Passenger.objects.all().count()                # 1309
Passenger.objects.count()                      # 1309

# aggregate(Count())는 결과를 사전 형식으로 반환
Passenger.objects.all().aggregate(Count('id')) # {'id__count': 1309}
Passenger.objects.aggregate(Count('id'))       # {'id__count': 1309}
Passenger.objects \
  .filter(age=0) \
  .aggregate(Count('id'))                      #  {'id__count': 263}

Passenger.objects.aggregate(Avg('age'))         # 평균 연령 {'age__avg': 23.877514667685258}
Passenger.objects.aggregate(Max('age'))         # 최대 연령 {'age__max': 80.0}

# 키 직접 지정 방식
Passenger.objects.aggregate(avg_age=Avg('age')) # 평균 연령 {'avg_age': 23.877514667685258}
Passenger.objects.aggregate(max_age=Max('age')) # 최대 연령 {'max_age': 80.0}

# 집계 내역 지정: (최대 - 평균)
Passenger.objects \
  .aggregate(age_diff=Max('age') - Avg('age'))
# {'age_diff': 56.12248533231474}

# 집계 내역 여러 개 지정
Passenger.objects \
  .aggregate(
    max_age_diff=Max('age') - Avg('age'),         # (최대 - 평균)
    min_age_diff=Min('age') - Avg('age'),         # (최소 - 평균)
  )
#{'max_age_diff': 56.12248533231474, 'min_age_diff': -23.877514667685258}
```
- 참고: 줄바꿈 문자 `'\'` 뒤에 주석 등을 쓰면 오류

<b>4.2.2 annotate() 함수로 (GROUP BY) 집계</b>
1. annotate 구문 형식
```Python {.line-numbers}
모델이름.objects \
  .values('구분_열') \
  .annotate(집계변수=집계함수('집계_열')) \
  .order_by('정렬_기준')
```
2. 승선_항구별로 승객 인원 집계하는 예제
```Python {.line-numbers}
# 승선_항구별로 승객 인원 집계 (승선_항구로 정렬)
Passenger.objects \
  .values('embarked') \
  .annotate(count_by_embarked=Count('embarked')) \
  .order_by('embarked')               # 승선_항구로 정렬
#  <QuerySet [
#    {'embarked': '', 'count_by_embarked': 2},
#    {'embarked': 'C', 'count_by_embarked': 270},
#    {'embarked': 'Q', 'count_by_embarked': 123},
#    {'embarked': 'S', 'count_by_embarked': 914}
#  ]>

# 승선_항구별로 승객 인원 집계 (승선 인원으로 정렬)
Passenger.objects \
  .values('embarked') \
  .annotate(count_by_embarked=Count('embarked')) \
  .order_by('count_by_embarked')      # 숭선 인원으로 정렬
#  <QuerySet [
#    {'embarked': '', 'count_by_embarked': 2},
#    {'embarked': 'Q', 'count_by_embarked': 123},
#    {'embarked': 'C', 'count_by_embarked': 270},
#    {'embarked': 'S', 'count_by_embarked': 914}
#  ]>

# 승선_항구로 승객 인원 집계 (널 항목 제거하고)
Passenger.objects \
  .values('embarked') \
  .exclude(embarked='') \
  .annotate(count_by_embarked=Count('embarked')) \
  .order_by('count_by_embarked')
#  <QuerySet [
#    {'embarked': 'Q', 'count_by_embarked': 123},
#    {'embarked': 'C', 'count_by_embarked': 270},
#    {'embarked': 'S', 'count_by_embarked': 914}
#  ]>
```
3. 티켓_등급별로 승객 인원 집계하는 예제
```Python {.line-numbers}
# 티켓_등급별 승객 인원 집계
Passenger.objects \
  .values('ticket_class') \
  .annotate(count_by_ticket_class=Count('ticket_class')) \
  .order_by('ticket_class')
#  <QuerySet [
#    {'ticket_class': 1, 'count_by_ticket_class': 323},
#    {'ticket_class': 2, 'count_by_ticket_class': 277},
#    {'ticket_class': 3, 'count_by_ticket_class': 709}
#  ]>
```
4. (티켓_등급, 생존_여부)별로 승객 인원 집계하는 예제
- (티켓_등급, 생존여부)별로 승객 인원 집계하는 예제-1
```Python {.line-numbers}
# Count('id')에 주목!
qs = Passenger.objects \
  .values('ticket_class', 'survived') \
  .annotate(cnt_by_tc=Count('id')) \
  .order_by('ticket_class', 'survived')
qs
#  <QuerySet [
#    {'ticket_class': 1, 'survived': False, 'cnt_by_tc': 123},
#    {'ticket_class': 1, 'survived': True, 'cnt_by_tc': 200},
#    {'ticket_class': 2, 'survived': False, 'cnt_by_tc': 158},
#    {'ticket_class': 2, 'survived': True, 'cnt_by_tc': 119},
#    {'ticket_class': 3, 'survived': False, 'cnt_by_tc': 528},
#    {'ticket_class': 3, 'survived': True, 'cnt_by_tc': 181}
#  ]>
# 쿼리셋이 위와 같이 구성되면 특정 항목에 직관적으로 접근하기 어려움
# (티켓_등급별로 리스트 내부 항목이 구분되지 않음)

str(qs.query)  # ORM 쿼리가 처리하는 내부적 SQL 문을 확인
# SELECT "ticket_class", "survived", COUNT("id") AS "cnt_by_tc_srvd"
# FROM "passengers_passenger"
# GROUP BY "ticket_class", "survived"
# ORDER BY "ticket_class" ASC, "survived" ASC
```
- (티켓_등급, 생존여부)별로 승객 인원 집계하는 예제-2
```Python {.line-numbers}
# filter 인자와 Q 객체를 활용 (위와 달리 티켓_등급별로 리스트 내부 항목이 구분됨)
qs = Passenger.objects \
  .values('ticket_class') \
  .annotate(
    survived_count=Count('ticket_class', filter=Q(survived=True)),
    not_survived_count=Count('ticket_class', filter=Q(survived=False))) \
  .order_by('ticket_class')
qs
#  <QuerySet [
#    {'ticket_class': 1, 'survived_count': 200, 'not_survived_count': 123},
#    {'ticket_class': 2, 'survived_count': 119, 'not_survived_count': 158},
#    {'ticket_class': 3, 'survived_count': 181, 'not_survived_count': 528}
#  ]>
# 티켓_등급별로 리스트 내부 항목이 구분되므로, `사전[키]` 형식으로 쉽게 접근 가능함
for d in qs:
  ticket_class = d['ticket_class']
  rate = d['survived_count'] \
      / (d['survived_count'] + d['not_survived_count']) * 100.0
  my_str = f'{ticket_class} 등급 티켓 승객의 생존율은 {rate:.1f} %'
  print(my_str)
#  1 등급 티켓 승객의 생존율은 61.9 %
#  2 등급 티켓 승객의 생존율은 43.0 %
#  3 등급 티켓 승객의 생존율은 25.5 %

# [f-string in python](https://bluese05.tistory.com/70)
```
5. filter와 Q 활용 연습
```Python {.line-numbers}
# (1) 성별로 인원 집계(집계 변수 이름 자동 지정)
count_passenger = Passenger.objects \
  .values('sex') \
  .annotate(Count('id')) \
  .order_by('sex')
# <QuerySet [
#  {'sex': 'F', 'id__count': 466},
#  {'sex': 'M', 'id__count': 843}
# ]>
```
```Python {.line-numbers}
# (2-1) 성별로 생존_여부 합계 집계 (BOOL 값: 0/1)
count_survived1 = Passenger.objects \
  .values('sex') \
  .annotate(sum_survived=Sum('survived')) \
  .order_by('sex')
#  <QuerySet [
#    {'sex': 'F', 'sum_survived': 339},
#    {'sex': 'M', 'sum_survived': 161}
#  ]>
```
```Python {.line-numbers}
# (2-2) 성별로 생존 여부 인원 집계
count_survived2 = Passenger.objects \
  .values('sex') \
  .annotate(
    survived_count=Count('sex', filter=Q(survived=True)),
    not_survived_count=Count('sex', filter=Q(survived=False))) \
  .order_by('sex')
#  <QuerySet [
#    {'sex': 'F', 'survived_count': 339, 'not_survived_count': 127},
#    {'sex': 'M', 'survived_count': 161, 'not_survived_count': 682}
#  ]>
# 생존율: 여성은 72.7%, 남성은 19.1% 
# SQL에서 count(*)는 널값을 포함하여 카운트
# SQL에서 count(열이름)은 널값을 제외하고 카운트
# SQL에서 빈 문자열 ''은 널값이 아니므로 카운트 결과에 포함됨
# 일반적으로, Count('sex')보다는 Count('id')로 쓰는 편을 권장함
```
```Python {.line-numbers}
# (3) 성별로 생존 비율 집계 ((1) 및 (2-2) 결과를 활용하여)
FEMALE = 0  # 상수 정의
MALE = 1    # 상수 정의
survival_rate_female = count_survived2[FEMALE]['survived_count'] \
  / count_passenger[FEMALE]['id__count'] \
  * 100
survival_rate_male = count_survived2[MALE]['survived_count'] \
  / count_passenger[MALE]['id__count'] \
  * 100
survival_rate_female  # 72.7
survival_rate_male    # 19.1
```
```Python {.line-numbers}
# (4) 성별 생존 비율을 한 방에 집계
from django.db.models import FloatField

FEMALE = 0  # 상수 정의
MALE = 1    # 상수 정의

qs = Passenger.objects \
  .values('sex') \
  .annotate(
    survived_rate=
      (1.0 * Count('id', filter=Q(survived=True), output_field=FloatField()))
      / (1.0 * Count('id', output_field=FloatField())) * 100.0,
    not_survived_rate=
      (1.0 * Count('id', filter=Q(survived=False), output_field=FloatField()))
      / (1.0 * Count('id', output_field=FloatField())) * 100.0
  ) \
  .order_by('sex')
# 위 코드에서 1.0을 곱하는 부분을 빼면, 모든 비율이 0.0으로 계산됨
# 위 코드에서 output_field 지정을 빼면, 모든 비율이 정수로 계산됨
# 위 코드에서 이들 둘을 모두 다 빼면, 모든 비율이 정수 0으로 계산됨

qs
# <QuerySet [
#   {'sex': 'F',
#    'survived_rate': 72.74678111587983,
#    'not_survived_rate': 27.253218884120173
#   },
#   {'sex': 'M',
#    'survived_rate': 19.098457888493474,
#    'not_survived_rate': 80.90154211150652
#   }
# ]>

qs[FEMALE]['survived_rate']  # 여성 생존율 72.7%
qs[MALE]['survived_rate']    # 남성 생존율 19.1%

str(qs.query)
# SELECT "sex",
#   (((1.0 * COUNT(CASE WHEN "survived" = True THEN "id" ELSE NULL END))
#     / (1.0 * COUNT("id"))) * 100.0) AS "survived_rate",
#   (((1.0 * COUNT(CASE WHEN "survived" = False THEN "id" ELSE NULL END))
#     / (1.0 * COUNT("id"))) * 100.0) AS "not_survived_rate"
# FROM "passengers_passenger"
# GROUP BY "sex"
# ORDER BY "sex" ASC
```

## 5. 차트 작성을 위한 DB 집계
### 5.1 SQL 집계 방식

- 테이블 구조 확인
```SQL {.line-numbers}
SELECT *
FROM sqlite_master
WHERE tbl_name='chart_passenger';
```
<table border="1" style="border-collapse:collapse">
  <tr>
    <th>type</th>
    <th>name</th>
    <th>tbl_name</th>
    <th>rootpage</th>
    <th>sql</th>
  </tr>
<tr>
  <td>table</td>
  <td>chart_passenger</td>
  <td>chart_passenger</td>
  <td>33</td>
  <td>CREATE TABLE &quot;chart_passenger&quot; </br>
      (&quot;id&quot; integer NOT NULL PRIMARY KEY AUTOINCREMENT, </br>
      &quot;name&quot; varchar(100) NOT NULL, </br>
      &quot;sex&quot; varchar(1) NOT NULL, </br>
      &quot;survived&quot; bool NOT NULL, </br>
      &quot;age&quot; real NULL, </br>
      &quot;ticket_class&quot; smallint unsigned NOT NULL, </br>
      &quot;embarked&quot; varchar(1) NOT NULL)
  </td>
</tr>
</table>
```

- 첫 5 행 출력
```SQL {.line-numbers}
SELECT * FROM chart_passenger LIMIT 5;
```
<table border="1" style="border-collapse:collapse">
  <tr><th>id</th><th>name</th><th>sex</th><th>survived</th><th>age</th><th>ticket_class</th><th>embarked</th></tr>
  <tr><td>1</td><td>Allen, Miss. Elisabeth Walton</td><td>F</td><td>1</td><td>29</td><td>1</td><td>S</td></tr>
  <tr><td>2</td><td>Allison, Master. Hudson Trevor</td><td>M</td><td>1</td><td>0.9167</td><td>1</td><td>S</td></tr>
  <tr><td>3</td><td>Allison, Miss. Helen Loraine</td><td>F</td><td>0</td><td>2</td><td>1</td><td>S</td></tr>
  <tr><td>4</td><td>Allison, Mr. Hudson Joshua Creighton</td><td>M</td><td>0</td><td>30</td><td>1</td><td>S</td></tr>
  <tr><td>5</td><td>Allison, Mrs. Hudson J C (Bessie Waldo Daniels)</td><td>F</td><td>0</td><td>25</td><td>1</td><td>S</td></tr>
</table>
- 전체 행 개수
```SQL {.line-numbers}
SELECT count() FROM chart_passenger;
```
<table border="1" style="border-collapse:collapse">
<tr><th>COUNT()</th></tr>
<tr><td>1309</td></tr>
</table>

<b>5.1.1 승선_항구별 인원 집계</b>
```SQL {.line-numbers}
SELECT embarked, count()
FROM chart_passenger
GROUP BY embarked;
```
<table border="1" style="border-collapse:collapse">
  <tr><th>embarked</th><th>count(embarked)</th></tr>
  <tr><td></td><td>2</td></tr>
  <tr><td>C</td><td>270</td></tr>
  <tr><td>Q</td><td>123</td></tr>
  <tr><td>S</td><td>914</td></tr>
</table>

<b>5.1.2 (티켓_등급, 생존_여부)별 인원집계</b>
```SQL {.line-numbers}
SELECT ticket_class, survived, count()
FROM chart_passenger
GROUP BY ticket_class, survived;
```
<table border="1" style="border-collapse:collapse">
<tr><th>ticket_class</th><th>survived</th><th>count()</th></tr>
<tr><td>1</td><td>0</td><td>123</td></tr>
<tr><td>1</td><td>1</td><td>200</td></tr>
<tr><td>2</td><td>0</td><td>158</td></tr>
<tr><td>2</td><td>1</td><td>119</td></tr>
<tr><td>3</td><td>0</td><td>528</td></tr>
<tr><td>3</td><td>1</td><td>181</td></tr>
</table>

### 5.2 ORM 집계 방식

<b>5.2.1 승선_항구별 인원 집계</b>
```SQL {.line-numbers}
SELECT embarked, count()
FROM chart_passenger
GROUP BY embarked;
```
```PYTHON {.line-numbers}
from django.db.models import Count

Passenger.objects \
  .values('embarked') \           # 특정 열 지정하여 {키: 값} 형태 사전을 획득
  .exclude(embarked='') \         # 특정 행 제외, 널 값인 행을 제외
  .annotate(total=Count('id')) \  # {키: 값} = {'total': 값) 형태 사전으로 집계
  .order_by('embarked')
#  [
#    {'embarked': 'C', 'total': 270},
#    {'embarked': 'Q', 'total': 123},
#    {'embarked': 'S', 'total': 914}
#  ]
```

<b>5.2.2 (티켓_등급, 생존_여부)별 인원 집계</b>
```SQL {.line-numbers}
SELECT ticket_class, survived, count()
FROM chart_passenger
GROUP BY ticket_class, survived;
```
```PYTHON {.line-numbers}
from django.db.models import Count, Q

Passenger.objects
    .values('ticket_class')
    .annotate(
        survived_count=Count('ticket_class', filter=Q(survived=True)),
        not_survived_count=Count('ticket_class', filter=Q(survived=False)))
    .order_by('ticket_class')
#  [
#    {'ticket_class': 1, 'survived_count': 200, 'not_survived_count': 123},
#    {'ticket_class': 2, 'survived_count': 119, 'not_survived_count': 158},
#    {'ticket_class': 3, 'survived_count': 181, 'not_survived_count': 528}
#  ]
```

## 6. 장고에서 차트 작성
- 네 가지 방법을 소개
  - 방법-1: 템플릿에서 데이터를 처리하는 방식
  - 방법-2: 뷰에서 데이터를 처리하는 방식
  - 방법-3: 뷰에서 모든 데이터 처리 작업을 수행하는 방식
  - 방법-4: 비동기 Ajax 통신을 이용하는 방법
  - 템플릿 내부에 복잡한 코드를 작성하는 방식은 권장하지 않음
- 접속 경로 추가
```PYTHON {.line-numbers}
# hChart/urls.py
from django.contrib import admin
from django.urls import path
from chart import views                                     # !!!

urlpatterns = [
    path('', views.home, name='home'),
    path('ticket-class/1/',
         views.ticket_class_view_1, name='ticket_class_view_1'),
    path('ticket-class/2/',
         views.ticket_class_view_2, name='ticket_class_view_2'),
    path('ticket-class/3/',
         views.ticket_class_view_3, name='ticket_class_view_3'),
    path('world-population/',
         views.world_population, name='world_population'),  # !!!
    path('json-example/', views.json_example, name='json_example'),
    path('json-example/data/', views.chart_data, name='chart_data'),
    path('admin/', admin.site.urls),
]
```
- 뷰 수정
```PYTHON {.line-numbers}
# chart/views.py
from django.shortcuts import render
from .models import Passenger
from django.db.models import Count, Q

def home(request):
    return render(request, 'home.html')

def world_population(request):
    return render(request, 'world_population.html')

def ticket_class_view_1(request):  # 방법 1
    pass

def ticket_class_view_2(request):  # 방법 2
    pass

def ticket_class_view_3(request):  # 방법 3
    pass

def json_example(request):  # 방법 4
    pass

def chart_data(request):  # 방법 4
    pass
```
- 홈 템플릿 추가
```HTML {.line-numbers}
<!--templates/home.html-->
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Django Highcharts Example</title>
</head>
<body>
  <ul>
    <li><a href="{% url 'world_population' %}">World Population</a></li>
    <li><a href="{% url 'ticket_class_view_1' %}">Ticket Class #1</a></li>
    <li><a href="{% url 'ticket_class_view_2' %}">Ticket Class #2</a></li>
    <li><a href="{% url 'ticket_class_view_3' %}">Ticket Class #3</a></li>
    <li><a href="{% url 'json_example' %}">JSON Example</a></li>
  </ul>

  <p>예제 코드는 <a href="https://simpleisbetterthancomplex.com/tutorial/2018/04/03/how-to-integrate-highcharts-js-with-django.html">How to Integrate Highcharts.js with Django</a> 기사에서 가져왔습니다.</p>
  <p>예제 코드는 <a href="https://github.com/sibtc/django-highcharts-example">GitHub</a>에서 다운로드 가능합니다.</p>
</body>
</html>
```
### 6.1 방법-1) 템플릿에서 데이터 처리
- 뷰 코드에서 ticket_class_view_1() 수정
```PYTHON {.line-numbers}
# chart/views.py
# ...
def ticket_class_view_1(request):  # 방법 1
    dataset = Passenger.objects \
        .values('ticket_class') \
        .annotate(
            survived_count=Count('ticket_class',
                                 filter=Q(survived=True)),
            not_survived_count=Count('ticket_class',
                                     filter=Q(survived=False))) \
        .order_by('ticket_class')
    return render(request, 'ticket_class_1.html', {'dataset': dataset})
#  dataset = [
#    {'ticket_class': 1, 'survived_count': 200, 'not_survived_count': 123},
#    {'ticket_class': 2, 'survived_count': 119, 'not_survived_count': 158},
#    {'ticket_class': 3, 'survived_count': 181, 'not_survived_count': 528}
#  ]
```
- ticket_class_1.html 추가
  템플릿에서 자바스크립트 태그 내부에 데이터 처리 코드를 작성
```HTML {.line-numbers}
<!--templates/ticket_class_1.html-->
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <title>Django Highcharts Example</title>
</head>
<body>
  <a href="{% url 'home' %}">Return to homepage</a>
  <div id="container"></div>
  <script src="https://code.highcharts.com/highcharts.src.js"></script>
  <script>
    Highcharts.chart('container', {
        chart: {
            type: 'column'
        },
        title: {
            text: 'Titanic Survivors by Ticket Class'
        },
        xAxis: {
            categories: [
              {% for entry in dataset %}
                '{{ entry.ticket_class }} Class'{% if not forloop.last %}, {% endif %}
              {% endfor %}
            ]
        },
        series: [{
            name: 'Survived',
            data: [
              {% for entry in dataset %}
                {{ entry.survived_count }}{% if not forloop.last %}, {% endif %}
              {% endfor %}
            ],
            color: 'green'
        }, {
            name: 'Not survived',
            data: [
              {% for entry in dataset %}
                {{ entry.not_survived_count }}{% if not forloop.last %}, {% endif %}
              {% endfor %}
            ],
            color: 'red'
        }]
    });
  </script>
</body>
</html>
```
- 실행 테스트
![](https://simpleisbetterthancomplex.com/media/2018/04/survivors-by-class.png)
- (템플릿에서 데이터를 처리하는) 이런 방법은 좋은 방식이 아님
- 더 좋은 방법은 뷰에서 데이터를 처리하는 방식

### 6.2 방법-2) 뷰에서 데이터 처리
- 뷰 코드에서 ticket_class_view_2() 수정
```PYTHON {.line-numbers}
# chart/views.py
import json  # ***json 임포트 추가***
# ...

def ticket_class_view_2(request):  # 방법 2
    dataset = Passenger.objects \
        .values('ticket_class') \
        .annotate(survived_count=Count('ticket_class', filter=Q(survived=True)),
                  not_survived_count=Count('ticket_class', filter=Q(survived=False))) \
        .order_by('ticket_class')

    # 빈 리스트 3종 준비
    categories = list()             # for xAxis
    survived_series = list()        # for series named 'Survived'
    not_survived_series = list()    # for series named 'Not survived'

    # 리스트 3종에 형식화된 값을 등록
    for entry in dataset:
        categories.append('%s Class' % entry['ticket_class'])    # for xAxis
        survived_series.append(entry['survived_count'])          # for series named 'Survived'
        not_survived_series.append(entry['not_survived_count'])  # for series named 'Not survived'

    # json.dumps() 함수로 리스트 3종을 JSON 데이터 형식으로 반환
    return render(request, 'ticket_class_2.html', {
        'categories': json.dumps(categories),
        'survived_series': json.dumps(survived_series),
        'not_survived_series': json.dumps(not_survived_series)
    })
# ...
```
- ticket_class_2.html 추가
  - `categories`를 바르게 렌더하기 위하여 `safe` 템플릿 필터를 적용하였음
    - 장고는 보안을 위하여 자동적으로 `'` 및 `"` 등의 문자를 이스케이프 처리함
    - 장고에게 내가 작성한 코드를 신뢰하고 있는 그대로 렌더하라고 지시
```HTML {.line-numbers}
<!--templates/ticket_class_2.html-->
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
    <title>Django Highcharts Example</title>
</head>
<body>
    <a href="{% url 'home' %}">Return to homepage</a>
    <div id="container"></div>
    <script src="https://code.highcharts.com/highcharts.src.js"></script>
    <script>
        Highcharts.chart('container', {
            chart: {
                type: 'column'
            },
            title: {
                text: 'Titanic Survivors by Ticket Class'
            },
            xAxis: {
                categories: {{ categories|safe }}  /* safe 필터 */
            },
            series: [{
                name: 'Survived',
                data: {{ survived_series }},
                color: 'green'
            }, {
                name: 'Not survived',
                data: {{ not_survived_series }},
                color: 'red'
            }]
        });
    </script>
</body>
</html>
```
- 실행 테스트
- 이 모든 작업을 백엔드에서 처리할 수도 있음

### 6.3 방법-3) 뷰에서 모든 데이터 처리
- 뷰 코드에서 ticket_class_view_3() 수정
```PYTHON {.line-numbers}
# chart/views.py
# ...

def ticket_class_view_3(request):  # 방법 3
    dataset = Passenger.objects \
        .values('ticket_class') \
        .annotate(survived_count=Count('ticket_class', filter=Q(survived=True)),
                  not_survived_count=Count('ticket_class', filter=Q(survived=False))) \
        .order_by('ticket_class')

    # 빈 리스트 3종 준비 (series 이름 뒤에 '_data' 추가)
    categories = list()                 # for xAxis
    survived_series_data = list()       # for series named 'Survived'
    not_survived_series_data = list()   # for series named 'Not survived'

    # 리스트 3종에 형식화된 값을 등록
    for entry in dataset:
        categories.append('%s Class' % entry['ticket_class'])         # for xAxis
        survived_series_data.append(entry['survived_count'])          # for series named 'Survived'
        not_survived_series_data.append(entry['not_survived_count'])  # for series named 'Not survived'

    survived_series = {
        'name': 'Survived',
        'data': survived_series_data,
        'color': 'green'
    }
    not_survived_series = {
        'name': 'Survived',
        'data': not_survived_series_data,
        'color': 'red'
    }

    chart = {
        'chart': {'type': 'column'},
        'title': {'text': 'Titanic Survivors by Ticket Class'},
        'xAxis': {'categories': categories},
        'series': [survived_series, not_survived_series]
    }
    dump = json.dumps(chart)

    return render(request, 'ticket_class_3.html', {'chart': dump})
```
- ticket_class_3.html 추가
```HTML {.line-numbers}
<!--templates/ticket_class_3.html-->
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
    <title>Django Highcharts Example</title>
</head>
<body>
    <a href="{% url 'home' %}">Return to homepage</a>
    <div id="container"></div>
    <script src="https://code.highcharts.com/highcharts.src.js"></script>
    <script>
        Highcharts.chart('container', {{ chart|safe }});
    </script>
</body>
</html>
```
- 실행 테스트
- 지금까지는 동기 통신 방식을 이용했음
- 비동기 통신 방식을 적용할 수 있음

### 6.4 방법-4) 비동기 통신 이용 방법
- highcharts 및 기타 서버와 상호작용하는 자바스크립트 라이브러리를 활용할 때 추천하는 방식
- 차트를 렌더할 때 비동기 호출을 사용하고, 서버에서 `JsonResponse`를 반환하는 방식
- 승선_항구별 승객 수 자료로 차트 작성
- views.py
  - `views.json_example`은 단순히 `json_example.html`을 반환
  - `views.chart_data`는 모든 힘든 일을 담당
    - 데이터베이스 조회
    - `chart` 사전 구성
- 뷰 코드에서 json_example() 및 chart_data() 수정
```PYTHON {.line-numbers}
# chart/views.py
# ...
from django.http import JsonResponse  # for chart_data()

# ...

def json_example(request):  # 접속 경로 'json-example/'에 대응하는 뷰
    return render(request, 'json_example.html')


def chart_data(request):  # 접속 경로 'json-example/data/'에 대응하는 뷰
    dataset = Passenger.objects \
        .values('embarked') \
        .exclude(embarked='') \
        .annotate(total=Count('id')) \
        .order_by('-total')
    #  [
    #    {'embarked': 'S', 'total': 914}
    #    {'embarked': 'C', 'total': 270},
    #    {'embarked': 'Q', 'total': 123},
    #  ]

    # # 탑승_항구 상수 정의
    # CHERBOURG = 'C'
    # QUEENSTOWN = 'Q'
    # SOUTHAMPTON = 'S'
    # PORT_CHOICES = (
    #     (CHERBOURG, 'Cherbourg'),
    #     (QUEENSTOWN, 'Queenstown'),
    #     (SOUTHAMPTON, 'Southampton'),
    # )
    port_display_name = dict()
    for port_tuple in Passenger.PORT_CHOICES:
        port_display_name[port_tuple[0]] = port_tuple[1]
    # port_display_name = {'C': 'Cherbourg', 'Q': 'Queenstown', 'S': 'Southampton'}

    chart = {
        'chart': {'type': 'pie'},
        'title': {'text': 'Number of Titanic Passengers by Embarkation Port'},
        'series': [{
            'name': 'Embarkation Port',
            'data': list(map(
                lambda row: {'name': port_display_name[row['embarked']], 'y': row['total']},
                dataset))
            # 'data': [ {'name': 'Southampton', 'y': 914},
            #           {'name': 'Cherbourg', 'y': 270},
            #           {'name': 'Queenstown', 'y': 123}]
        }]
    }
    # [list(map(lambda))](https://wikidocs.net/64)

    return JsonResponse(chart)
```
- json_example.html
  - `div#container`는 차트가 그려질 영역
  - `data-url="{% url 'chart_data' %}"` 부분을 보면,
      - 결국 'chart_data'라는 이름의 접속경로로 연결되고,
      - 최종적으로 views.chart_data() 함수로 연결되어,
      - `JsonResponse(chart)`를 반환받게 됨
  - jQuery 코드 `$.ajax()`에 연결됨
    - `url`로 지정된 $("#container").attr("data-url")로부터
    - `dataType`으로 지정된 JSON 객체의 반환을 대기하라는 ajax 요청을 지시함
    - ajax 요청에 대한 응답이 완료되면,
      JSON 응답이 `success` 함수의 `data` 매개변수 내부로 전달됨
    - 최종적으로, `success` 함수 내부에서 Highcharts API를 활용하여 차트를 그리게 됨
```HTML {.line-numbers}
<!--templates/json_example.html-->
<!doctype html>
<html>
<head>
      <meta charset="utf-8">
      <title>Django Highcharts Example</title>
</head>
<body>
    <a href="{% url 'home' %}">Return to homepage</a>
    <div id="container" data-url="{% url 'chart_data' %}"></div>
    <script src="https://code.highcharts.com/highcharts.src.js"></script>
    <script src="https://code.jquery.com/jquery-3.3.1.min.js"></script>
    <script>
    $.ajax({
        url: $("#container").attr("data-url"),
        dataType: 'json',
        success: function(data) {
            Highcharts.chart("container", data);
        }
    });
    </script>
</body>
</html>
```
- 실행 테스트
![](https://simpleisbetterthancomplex.com/media/2018/04/highcharts-pie-chart.png)

## 7. 결론
- `Highcharts.js`와 `Django`의 통합
  - 이러한 통합 방식은 `Charts.js`와 같은 다른 차트 라이브러리에 대하여도 공통적으로 적용됨
  - 장고 템플릿 언어를 통하여 자바스크립트 코드와 상호작용하는 방식은 바람직하지 않음
  - JSON 객체 형식의 데이터를 활용하는 방식을 강추함
- 차트 작성에 적합한 형식으로 데이터를 공급하는 방법
  - 우선, 정적 자료를 하드코딩 방식으로 차트를 그려봄으로써 필요한 데이터 형식을 파악
  - 이후, 파이썬 터미널에서 QuerySet을 작성
  - 최종적으로 뷰 함수를 작성
- 과제
  - 데이터 시각화 작업을 수행했던 CCTV 및 범죄 자료에 대해 적용을 시도
  - 이러한 자료를 처리할 때, DB 적재 작업이 적절한지 고민이 필요함
  - csv 파일의 데이터로 직접 차트를 작성할 수도 있음
  - 그러나 웹 사이트 운영 과정에서 생성되어 DB에 적재되어 있는 자료라면, ...
- 추가 학습 자료
  - [How to Integrate Highcharts.js with Django by Vitor Freitas](https://simpleisbetterthancomplex.com/tutorial/2018/04/03/how-to-integrate-highcharts-js-with-django.html)
  - [django-highcharts-example repository](https://github.com/sibtc/django-highcharts-example) 복제 및 실행
  - [titanic.csv](https://github.com/sibtc/django-highcharts-example/blob/master/titanic.csv) 자료 형식
  - [Passenger model class](https://github.com/sibtc/django-highcharts-example/blob/master/passengers/models.py) 이해
  - [Highcharts.js demo page](https://www.highcharts.com/demo)에서 원하는 차트 형태 선별
  - [list(map(lambda))](https://wikidocs.net/64)

![](http://logistex2020.pythonanywhere.com/media/photos/2020/05/30/Opera_%EC%8A%A4%EB%83%85%EC%83%B7_2020-05-30_200324_logistex2020.pythonanywhere.com.png)