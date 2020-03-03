# Relayo

## Install Python Packages
```
$ cd relayo
$ virtualenv env
$ source env/bin/activate
$ pip install -r requirements/development.txt
```
If `core/utils.c:3866:18: error: passing an object that undergoes default argument promotion to 'va_start' has undefined behavior [-Werror,-Wvarargs]` error raise on 'pip install', change uwsgi version: 2.0.11.2 -> 2.0.15 (it is only development env on mac 10.14.1)

<br>

## Create Database
```
$ createdb relayo -E UTF8
```

<br>

## Run Migration

Before run migration, run below command first on fabric
```
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

NOTE: `local_only updateconf` : copy config files under local `conf` directory

```
$ cd relayo
$ RELAYO_RUN_ENV=local python daemon/manage.py migrate
```

If `django.contrib.gis.geos.error.GEOSException: Could not parse version info string "3.6.2-CAPI-1.10.2 4d2925d6"` error raise on migration,
modify 'libgeos.py'
```
$ emacs ~/.virtualenvs/relayo/lib/python2.7/site-packages/django/contrib/gis/geos/libgeos.py
```
fix 'geos_version() -> geos_version().split(' ')[0]'

<br>

## Create local/development configurations
```
$ fab local_only updateconf
```

<br>

## Run Daemon
```
$ cd relayo
$ RELAYO_RUN_ENV=local python daemon/manage.py runserver
```

NOTE: Try to change combu 3.0.26 to 3.0.30 if kombu error raise on execute (only dev env)

<br>

## Run Celery Workers
```
$ cd relayo
$ DJANGO_SETTINGS_MODULE=daemon.main.settings RELAYO_RUN_ENV=local python -m workers.app worker -Q q_order_new
```

With logging,
```
$ export RELAYO_RUN_ENV=local
$ export LOGGER_LOG_LEVEL=DEBUG
$ python -m workers.app worker -l info
```

To enable file logging, set following env variables.
```
export LOGGER_LOG_PATH=/path/to/log_file.log
export LOGGER_VERBOSE_LOG_PATH=/path/to/verbose_log_file.log
```

To enable rsyslog, you need to prepare log path with the proper permission.
```
$ mkdir -p /opt/relayo
$ chown syslog:adm -R /opt/relayo
```

And then, set env `LOGGER_USE_SYSLOG` to `true` before running celery workers.
```
$ export LOGGER_USE_SYSLOG=true
$ RELAYO_RUN_ENV=local python -m workers.app worker
```


For dedicated workers for queue `order`.
```
$ RELAYO_RUN_ENV=local python -m workers.app worker -Q q_order_new
```

<br>

## Deployment (Fabric)

#### Fabric Command Lists

  - Deployment Targets: `production`, `staging`

  - Command Types: `update`, `cutover`

#### Process

  Can combine above target, server type and command type

  1. `update` : make target server fetch the latest and go to head, then reload server process

  2. `cutover` : reload nginx @ target with next environment

  3. `updateconf` : upload config files in `fabfile/confs/{target}` to target hosts

  4. `local_only updateconf` : copy config files under local `conf` directory

  5. `migrate` : apply migrations

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
The exported class reference document would be located in "<Yor-project-directory>/relayo/docs/api/_build/html/index.html"
```
$ cd relayo
$ make doc
$ make cleandoc # to clean docs/api
```


## Relayo Management Commands

### Create API Client for POS vendor
```
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

<br>

## Relayo Redis

Redis: 5.0.3

NOTE: django-constance use db '0'
