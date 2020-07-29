# Conveyo


`난 하나를 썼어요.`
``난 두 개를 썼는데요?``

* Last modified date: 2020-07-21

**T&I 스쿼드의 MSA로서 SNS(Hubyo)를 통해 받은 주문을 전달 및 처리하는 역할을 한다.**

* Python: 3.7.*
* Web framework: Chalice
* Database: Redis


## 1. Poetry 설치

[Poetry](https://python-poetry.org/)는 패키지 매니저로 시스템에 설치 안 되어 있다면 먼저 설치한다. 설치되어 있다면 2번으로.

```shell
# 가장 권장되는 방법
$ curl -sSL https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py | python
```

* #### poetry의 실행파일의 위치를 shell 환경설정 파일을 통해 PATH에 추가한다.

```shell
export PATH=$PATH:$HOME/.poetry/bin
```


## 2. Conveyo 설치 및 가상환경 생성

* #### Repo 클론

```shell
cd $WORKDIR
git clone git@github.com:yogiyo/conveyo.git
cd conveyo
```

* #### pyenv로 파이썬 설치

현재 Conveyo의 파이썬 버전은 3.7로 로컬 파이썬 버전이 3.7대가 아니라면 [pyenv](https://github.com/pyenv/pyenv)를 설치하고 파이썬 버전을 관리하는 것을 추천한다.  
**pyenv는 복수의 파이썬 버전을 손쉽게 설치 & 전환할 수 있도록 도와주는 프로그램이다.**

conveyo의 파이썬 버전에 맞는 파이썬을 설치한다.

```shell
$ pyenv install 3.7.5  # micro version은 임의
```

* #### 가상환경 생성 및 패키지 설치

poetry는 패키지 관리뿐 아니라 가상환경 생성 및 관리도 지원한다. 여기서는 poetry에서 생성되는 가상환경을 쓴다.

pyenv를 사용해 파이썬을 로컬 -> 3.7로 전환해서 프로젝트에 맞는 파이썬 버전을 지목한다.  
poetry는 `pyproject.toml`에 지정된 파이썬 버전을 자동으로 찾아 가상환경을 생성하고 패키지를 설치한다.(`poetry.lock`이 있으면 대신 사용)  

```shell
$ pyenv shell 3.7.5  # 3.7.5 버전을 사용하도록 지목
$ poetry install     # 가상환경 생성 및 패키지 설치
```

* #### 가상환경을 로드한다.

```shell
$ poetry shell

Spawning shell within /Users/a202002010/Library/Caches/pypoetry/virtualenvs/conveyo-ptXzqI0E-py3.7

(conveyo-ptXzqI0E-py3.7) $
```

* #### pre-commit script 설치

pre-commit은 범용적인 precommit git hook manager로, conveyo는 pre-commit을 사용해 다양한 git hook 작업을 관리하고 있다.
pre-commit에 정의된 hook을 실제 git hook에 설치한다.

```shell
(conveyo-ptXzqI0E-py3.7) $ pre-commit install
```

이후로는 commit마다 작업분에 대해 정의된 git hook이 자동으로 실행된다.


## 3. Redis 컨테이너 로드

* #### Redis 도커 컨테이너를 로드한다.

```shell
$ cd local_dev
$ docker-compose up -d
```

* #### docker Redis가 제대로 연결되는지 확인한다.

local 환경일 때 Conveyo redis는 기존에 설치한 YGY redis 포트와 겹치지 않도록 다른 포트(6378)를 사용하고 있다.

```shell
(conveyo-ptXzqI0E-py3.7) $ python -c 'import redis; redis_instance = redis.Redis(host="127.0.0.1", port="6378"); print(redis_instance.ping())'

True
```


## 4. chalice 어플리케이션 local 환경에서 실행

Conveyo는 AWS Lambda를 사용해 서버리스 어플리케이션을 생성, 배포하는 파이썬 프레임워크인 `Chalice`를 사용한다.

* #### 로컬 환경에서 chalice 실행

`chalice local` 수행시 '--stage local' 을 추가하여 local stage를 사용한다.

```shell
(conveyo-ptXzqI0E-py3.7) $ chalice local --stage local
```


## 5. local 테스트

chalice를 로컬로 실행한 상태에서 다음 명령어들을 통해 동작상태를 확인한다.

* #### Simple ping test:

```shell
$ curl -X GET '127.0.0.1:8000/ping/'

pong
```

* #### 주문 상태 등록 예시:

```shell
$ curl -X POST -H "Content-Type: application/json" '127.0.0.1:8000/order-relay-status/' --data '{"origin": "Yogiyo", "target": "Relayo", "status_map": {"filter_expr": ".result.msg, .result.status", "values": ["ok", "success"]}, "status": "SUCCESS"}'

{"result": "Success"}
```

* #### 등록된 상태 조회:

```shell
$ curl -X GET -H "Content-Type: application/json" '127.0.0.1:8000/order-relay-status/?origin=Yogiyo&target=Relayo'

"origin": "Yogiyo", "target": "Relayo", "status_map": {".result.msg,.result.status": [[["ok", "success"], "SUCCESS"]]}}
```

* #### pytest 전체 테스트

```shell
(conveyo-ptXzqI0E-py3.7) $ poetry run pytest

rootdir: /Users/a2020202020/workspace/conveyo, inifile:
collected 61 items

chalicelib/hubyo/test_hubyo.py ..
...

==== 61 passed, 2 warnings in 0.83 seconds ===
```
