# Conveyo

* Last modified date: 2020-07-14

**T&I 스쿼드의 MSA로서 SNS(Hubyo)를 통해 받은 주문을 전달 및 처리하며 chalice로 작성되어 있다.**


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


## 2. Conveyo 및 가상환경 생성

* #### Repo 클론

```shell
cd $WORKDIR
git clone git@github.com:yogiyo/conveyo.git
cd conveyo
```

* #### pyenv로 가상환경 생성

현재 Conveyo의 파이썬 버전은 3.7로 로컬 파이썬 버전이 3.7대가 아니라면 [pyenv](https://github.com/pyenv/pyenv)를 설치하고 파이썬 버전을 관리하는 것을 추천한다.  
pyenv는 복수의 파이썬 버전을 손쉽게 switch 할 수 있도록 도와주는 프로그램으로, 가상환경 생성을 위해서는 추가로 [pyenv virtualenv](https://github.com/pyenv/pyenv-virtualenv)까지 설치해준다.  

이후 가상환경을 설정한다.

conveyo의 파이썬 버전에 맞는 가상환경을 생성한다.

```shell
$ pyenv install 3.7.5  # micro version은 임의
$ pyenv virtualenv 3.7.5 conveyo-venv
```

파이썬 binary path를 가상환경의 파이썬으로 맞춘 뒤 패키지를 설치한다. 이때 poetry용 가상환경이 생성된다.  
Poetry는 `pyproject.toml`에 지정된 파이썬 버전을 자동으로 찾아 가상환경을 생성하고 패키지를 설치한다.(`poetry.lock`이 있으면 대신 사용)  

```shell
$ pyenv shell conveyo-venv
$ poetry install


  The currently activated Python version 2.7.16 is not supported by the project (^3.7).
  Trying to find and use a compatible version.
  Using python3 (3.7.5)
  Creating virtualenv conveyo-ptXzqI0E-py3.7 in 특정경로
  Installing dependencies from lock file


  Package operations: 57 installs, 0 updates, 0 removals

    - Installing six (1.12.0)
    - Installing docutils (0.14)
  ...
```


## 3. Redis 컨테이너 로드

* #### Redis 도커 컨테이너를 로드한다.

```shell
$ cd local_dev
$ docker-compose up -d
```

* #### docker Redis가 제대로 연결되는지 확인한다.

local 환경일 때 Conveyo redis는 기존에 설치한 YGY redis 포트와 겹치지 않도록 다른 포트(6378)를 사용하고 있다.

```shell
$ poetry run python -c 'import redis; redis_instance = redis.Redis(host="127.0.0.1", port="6378"); print(redis_instance.ping())'

True
```


## 4. chalice 어플리케이션 local 환경에서 실행

Conveyo는 AWS Lambda를 사용해 서버리스 어플리케이션을 생성, 배포하는 파이썬 프레임워크인 `Chalice`를 사용한다.

* #### 로컬 환경에서 chalice 실행

`chalice local` 수행시 '--stage local' 을 추가하여 local stage를 사용한다.

```shell
$ poetry run chalice local --stage local
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
