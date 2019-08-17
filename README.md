# aleph-worker

Dokku Dockerfile deploy repo for an aleph worker

Note that logs are persistent across container restarts and are in `/var/log/aleph/celery.log`.

## Config

### Basics

# Aleph in Dokku

## Deployment

```
dokku config:set aleph \
    ALEPH_APP_TITLE="Aleph Code4SA" \
    ALEPH_APP_NAME=aleph \
    ALEPH_FAVICON=http://code4sa.org/favicon.ico \
    ALEPH_APP_URL=http://search.opengazettes.org.za \
    ALEPH_LOGO=http://code4sa.org/images/logo.png \
    ALEPH_SECRET_KEY=... \
    ALEPH_URL_SCHEME=http \
    ALEPH_OAUTH_KEY=... \
    ALEPH_OAUTH_SECRET=... \
    ALEPH_ARCHIVE_TYPE=s3 \
    ALEPH_ARCHIVE_BUCKET=code4sa-aleph \
    AWS_ACCESS_KEY_ID=... \
    AWS_SECRET_ACCESS_KEY=... \
    ALEPH_BROKER_URI=sqs://sqs.eu-west-1.amazonaws.com/.../
    ALEPH_DATABASE_URI=postgresql://aleph:aleph@postgres/aleph \
    ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200/
    C_FORCE_ROOT='true' \
    POLYGLOT_DATA_PATH=/opt/aleph/data \
    TESSDATA_PREFIX=/usr/share/tesseract-ocr \
    ALEPH_PDF_OCR_IMAGE_PAGES=false \
    CELERY_CONCURRENCY=4 \
    CELERYD_MAX_TASKS_PER_CHILD=1 \
    TIKA_URI='http://tika:9998/' \
    PDF_TEXT_MODULE='tika' \
    LOGLEVEL=DEBUG \
    ZA_GAZETTE_ARCHIVE_URI=https://archive.opengazettes.org.za/ \
    CELERY_RDBSIG=1
```

```
docker run -d --name=tika logicalspark/docker-tikaserver
```

```
dokku docker-options:add aleph-worker run,deploy  "-v /var/log/aleph:/var/log"
dokku docker-options:add aleph-worker run,deploy  "-v /var/lib/aleph:/opt/aleph/data"
dokku docker-options:add aleph-worker run,deploy  "--link tika"
```


To conserve resources, don't use zero-downtime deployment. Since this can
consume a lot of memory, we don't want two instances with all their workers
running concurrently and there aren't web users seeing downtime.

    dokku checks:disable aleph-worker

Tuning performance for the available resources is extremely important.
Since this is a CPU and RAM intensive task, you don't want to over-utilise
but compute is expensive to try to utilise well.

For a machine with _n_ cores you can spare for this work, it's probably
best to use _n_ workers

    CELERY_CONCURRENCY=4

Python processes apparently don't release memory very well, and this is
shelling out to other things like tesseract, so you have to cycle worker
processes. Start small, say at `CELERYD_MAX_TASKS_PER_CHILD=2` and raise
this carefully. Letting workers exhaust your memory will make your machine
unreachable indefinitely and you'll probably have to do a hard reset which
could make docker so upset you have to reinstall it.

On a machine with 8GB RAM and CELERY_CONCURRENCY=4 running nothing else,
I found `CELERYD_MAX_TASKS_PER_CHILD=5` to work well, utilising up to about
6GB while ingesting already-OCR'd gazette PDFs.

## Debugging / troubleshooting

To figure out what a worker process is doing, use Celery's rdb. Send SIGUSR2 to the process and connect to a pdb session using telnet, as described at http://docs.celeryproject.org/en/v2.3.3/tutorials/debugging.html#enabling-the-breakpoint-signal. This will pause execution until you continue it in pdb or exit pdb.