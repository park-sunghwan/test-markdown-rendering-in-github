# Relayo Onboarding process

* Last modified: 2020-06-18

## 1. Clone repository and install Python Packages

```bash
# Clone
$ cd $WORKDIR
$ git clone git@github.com:yogiyo/relayo.git
$ cd relayo

# Install packages
$ virtualenv env
$ source env/bin/activate
$ pip install -r requirements/development.txt
```

#### !!! You will encounter an error while installing requirements:

```
If `core/utils.c:3866:18: error: passing an object that undergoes default argument promotion to 'va_start' has undefined behavior [-Werror,-Wvarargs]` error raise on 'pip install', change uwsgi version: 2.0.11.2 -> 2.0.15 (it is only development env on mac 10.14.1)
```
As it reads, change `uwsgi` version to 2.0.15.(Change `requirements/base.txt`, install and turn it back again.)


## 2. Create Database
```bash
$ createdb relayo -E UTF8
```

## 3. Run Migration

Before run migration, you have to run some commands below first.

* Copy django settings files

```bash
$ fab local_only updateconf
[localhost] Executing task 'local_only'
[localhost] Executing task 'updateconf'
[localhost] local: cp fabfile/confs/local/daemon.py daemon/main/settings/local.py
[localhost] local: cp fabfile/confs/local/ygybe.py libs/helpers/ygybe/conf/local.py
[localhost] local: cp fabfile/confs/local/worker.py workers/conf/local.py
[localhost] local: cp fabfile/confs/staging/daemon.py daemon/main/settings/staging.py
[localhost] local: cp fabfile/confs/staging/ygybe.py libs/helpers/ygybe/conf/staging.py
[localhost] local: cp fabfile/confs/staging/worker.py workers/conf/staging.py
[localhost] local: cp fabfile/confs/test/daemon.py daemon/main/settings/test.py
[localhost] local: cp fabfile/confs/test/ygybe.py libs/helpers/ygybe/conf/test.py
[localhost] local: cp fabfile/confs/test/worker.py workers/conf/test.py
[localhost] local: cp fabfile/confs/production/daemon.py daemon/main/settings/production.py
[localhost] local: cp fabfile/confs/production/ygybe.py libs/helpers/ygybe/conf/production.py
[localhost] local: cp fabfile/confs/production/worker.py workers/conf/production.py

Done.
```

`local_only updateconf` command copies config files under local `fabfile/conf` directory to daemon/main/settings

* If you installed YGY to use targetyo with your employee number for targetyo platform, go to `daemon/main/settings/local.py` and change this part

```python
TARGETYO_PLATFORM_NAME = 'YGY-dev'  # Change it to your employee number if you use staging targetyo on local machine

# change this part as follows with your employee number:

TARGETYO_PLATFORM_NAME = 'YGY-A20202020'
```

* And run migrate

```bash
$ cd relayo
$ RELAYO_RUN_ENV=local python daemon/manage.py migrate
```

#### !!! You will encounter an error on migration

```
If `django.contrib.gis.geos.error.GEOSException: Could not parse version info string "3.6.2-CAPI-1.10.2 4d2925d6"` error raise on migration,
modify 'libgeos.py'
```

Open `{your-virtualenv-path}/lib/python2.7/site-packages/django/contrib/gis/geos/libgeos.py` and change line number 144 as follows:

```python
# before
ver = geos_version()

# to
ver = geos_version().split()[0]
```


## 4. Generate local environment variables

You have to set local environment variables to run local server(it works same as Yogiyo_Web)
Check it out `fabfile.__init__.py`

```bash
$ fab generate_local_env

[localhost] Executing task 'generate_local_env'
[localhost] Executing task 'generate_env'
Download local env to /Users/{your-number}/workspace/relayo/fabfile/../.env.new
[localhost] local: diff -uN /Users/{your-number}/workspace/relayo/fabfile/../.env /Users/{your-number}/workspace/relayo/fabfile/../.env.new || true
[localhost] local: mv /Users/{your-number}/workspace/relayo/fabfile/../.env.new /Users/{your_number}/workspace/relayo/fabfile/../.env
```


## 5. Run Daemon runserver
```bash
$ cd relayo
$ RELAYO_RUN_ENV=local python daemon/manage.py runserver
```
**Don't forget to set `RELAYO_RUN_ENV` env to `local` to run local server. You'd better set this variable in Pycharm too.**

NOTE: Try to update `kombu` from 3.0.26 to 3.0.30 if kombu error raise on execution.(only dev env)

```bash
$ pip install kombu==3.0.30
```

## 6. Run Celery Workers

```bash
$ cd $WORKDIR/relayo
$ DJANGO_SETTINGS_MODULE=daemon.main.settings RELAYO_RUN_ENV=local python -m workers.app worker -Q q_order_new
```

With logging,
```bash
$ export RELAYO_RUN_ENV=local
$ export LOGGER_LOG_LEVEL=DEBUG
$ python -m workers.app worker -l info
```

To enable file logging, set following env variables.
```bash
export LOGGER_LOG_PATH=/path/to/log_file.log
export LOGGER_VERBOSE_LOG_PATH=/path/to/verbose_log_file.log
```

To enable rsyslog, you need to prepare log path with the proper permission.
```bash
$ mkdir -p /opt/relayo
$ chown syslog:adm -R /opt/relayo
```

And then, set env `LOGGER_USE_SYSLOG` to `true` before running celery workers.
```bash
$ export LOGGER_USE_SYSLOG=true
$ RELAYO_RUN_ENV=local python -m workers.app worker
```


For dedicated workers for queue `order`.
```bash
$ RELAYO_RUN_ENV=local python -m workers.app worker -Q q_order_new
```

## 7. Deployment (Fabric)

#### Fabric Command Lists

  - Deployment Targets: `production`, `staging`

  - Command Types: `update`, `cutover`

#### Process

  Can combine above target, server type and command type

  * `update` : make target server fetch the latest and go to head, then reload server process

  * `cutover` : reload nginx @ target with next environment

  * `updateconf` : upload config files in `fabfile/confs/{target}` to target hosts

  * `local_only updateconf` : copy config files under local `conf` directory

  * `migrate` : apply migrations

#### Examples
  ```
  $ cd relayo
  $ fab staging:api update          # update API server @ staging
  $ fab staging:worker update       # update workers @ staging
  $ fab staging update              # update both servers @ staging
  $ fab staging:api cutover         # cutover API server @ staging
  $ fab staging:worker cutover      # cutover worker server @ staging
  $ fab staging cutover             # cutover API server @ staging
  # fab production update cutover   # update & cutover both servers @ production
  ```

### Create a class reference document
The exported class reference document would be located in `<Your-project-directory>/relayo/docs/api/_build/html/index.html`

```bash
$ cd relayo
$ make doc
$ make cleandoc # to clean docs/api
```

## 7. Checklist after onboarding

### Testing Relayo tests

After installation, **you can be sure that your onboarding on relayo is successful after passing daemon tests.**  
Let's check it out right now :)

```bash
$ cd $WORKDIR/relayo
$ RELAYO_RUN_ENV=test python daemon/manage.py test daemon --keepdb

...
...
Ran 192 tests in 18.742s

OK (skipped=14)
```

You should pass all relayo daemon tests except for skipped ones.  
**Remember, you have to set `RELAYO_RUN_ENV` to `test` for running tests. It applies same when setting pycharm testing configurations.**

Django creates test databases everytime you run tests and destroy them after the tests. `--keepdb` option preserves test databases and django doesn't recreate them to test next time.
But django sometimes ignores this option and create databases when:
  1. Running tests first time in repository.
  2. Any migrations need to be applied.


### Create API Client for POS vendor

```bash
$ cd relayo
$ python daemon/manage.py createapiclient

Franchise Name: BHC-Chiken
Franchise code: UNITAS-BHC
Franchise Callback URL (blank if not available): http://unifos.com/cb
Is POS responsible? (Y/n): Y
Does POS use terminal protocol? (y/N): N
+----------------+--------------------------------------------------+
| FIELD          | VALUE                                            |
+----------------+--------------------------------------------------+
| Franchise Name | BHC-Chiken                                       |
| Franchise Code | UNITAS-BHC                                       |
| API key        | aca2ecd958b04ca39283f6f5a949d06b                 |
| API Secret     | 17117ffd758e9f8dc88c135d2385323b9c34737cb51efb94 |
| Callback URL   | http://unifos.com/cb                             |
+----------------+--------------------------------------------------+
```

## Relayo Redis

Redis: 5.0.3

NOTE: django-constance use db '0'
