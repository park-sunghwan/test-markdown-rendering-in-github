# Conveyo

* Last modified date: 2020-07-10

T&I 스쿼드의 MSA로서 SNS(Hubyo)를 통해 받은 주문을 전달 및 처리하며 chalice로 작성되어 있다.


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


## 2. Conveyo 설치

* #### Repo 클론

```shell
cd $WORKDIR
git clone git@github.com:yogiyo/conveyo.git
cd conveyo
```

* #### 가상환경 생성 및 라이브러리 설치

Poetry는 `pyproject.toml`에 지정된 파이썬 버전을 자동으로 찾아 가상환경을 생성하고 패키지를 설치한다.(`poetry.lock`이 있으면 대신 사용)  
참고로 현재 conveyo의 파이썬 버전은 3.7로 향후 3.8로 업그레이드할 예정.  

```shell
$ poetry install


  The currently activated Python version 2.7.16 is not supported by the project (^3.7).
  Trying to find and use a compatible version.
  Using python3 (3.7.6)
  Creating virtualenv conveyo-ptXzqI0E-py3.7 in 특정경로
  Installing dependencies from lock file


  Package operations: 57 installs, 0 updates, 0 removals

    - Installing six (1.12.0)
    - Installing docutils (0.14)
  ...
```

## 3. direnv 설치

[direnv](https://direnv.net/)는 프로젝트(디렉토리)별로 환경변수를 설정/해제할 수 있는 도구로 현재 Redis 환경변수 등을 설정할 때 사용하고 있다.

```shell
$ brew install direnv
```

#### direnv hook을 걸어준다. 

```shell
eval "$(direnv hook zsh)" >> ~/.zshrc  # 쉘마다 조금씩 다르며 'https://direnv.net/docs/hook.html'를 참고
```

shell을 재시작한다.


## 3. Redis 설정

Conveyo는 데이터베이스로 Redis를 사용한다. 따라서 로컬에서 개발하고 테스트하기 위해서는 docker Redis 설정을 해줘야 한다.  
Conveyo 설치 전에 Yogiyo\_Web을 도커로 먼저 설치했을 것이고 그러면 YGY Redis 인스턴스가 이미 6379 Redis default 포트를 점유하고 있을 것이다. 따라서 Conveyo 용으로는 다른 포트를 사용한다.

* #### 코드에서 Redis instance를 호출할 때 사용할 환경변수를 설정한다.

conveyo 디렉토리의 `.envrc` 파일을 열어 다음을 추가한다.

```shell
# conveyo redis settings
export REDIS_HOST=127.0.0.1
export REDIS_PORT=6378
export REDIS_DB=0
```

* #### .envrc에 추가된 환경변수를 이 디렉토리 내에서 허용한다.

```shell
$ direnv allow .

direnv: reloading
direnv: loading .envrc
direnv export: ...
```

* #### Redis 도커 컨테이너를 로드한다.

```shell
$ cd local_dev
$ docker-compose up -d
```

* #### docker Redis가 제대로 연결되는지 확인한다.

```shell script
$ poetry run python -c 'import os, redis; redis_instance = redis.Redis(host=os.getenv("REDIS_HOST"), port=os.getenv("REDIS_PORT")); print(redis_instance.ping())'

True
```


## 4. chalice 어플리케이션 local 환경에서 실행

Conveyo는 웹어플리케이션으로 Django를 사용하지 않는다. 대신 AWS Lambda를 사용해 서버리스 어플리케이션을 생성, 배포하는 파이썬 프레임워크인 `Chalice`를 사용한다.

* #### 로컬 환경에서 chalice 실행

`chalice local` 수행시 '--stage local' 을 추가하여 local redis를 쓴다.

```shell
$ poetry run chalice local --stage local  # stage를 local로 설정
```


## 5. local 테스트

chalice를 로컬로 실행한 상태에서 다음 명령어들을 통해 동작상태를 확인한다.

* #### Simple ping test:

```shell script
$ curl -X GET '127.0.0.1:8000/ping/'

pong
```

* #### 주문 상태 등록 예시:

```shell
$ curl -X POST -H "Content-Type: application/json" '127.0.0.1:8000/order-relay-status/' --data '{"origin": "Yogiyo", "target": "Relayo", "status_map": {"filter_expr": ".result.msg, .result.status", "values": ["ok", "success"]}, "status": "SUCCESS"}'

{"result": "Success"}
```

* #### 등록된 상태 조회:

```shell script
$ curl -X GET -H "Content-Type: application/json" '127.0.0.1:8000/order-relay-status/?origin=Yogiyo&target=Relayo'

"origin": "Yogiyo", "target": "Relayo", "status_map": {".result.msg,.result.status": [[["ok", "success"], "SUCCESS"]]}}
```
