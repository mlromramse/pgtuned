# PGTuned

PGTuned is a first attempt at building a Docker PostgreSQL image that includes basic performance tuning based on available ressources and contemplated use-case.  
This project includes a bash script equivalent of [PGTune](https://github.com/le0pard/pgtune)

## `pgtune.sh`

This script is a bash port of [PGTune](https://github.com/le0pard/pgtune).

```
Usage: pgtune.sh [-h] [-v PG_VERSION] [-t DB_TYPE] [-m TOTAL_MEM] [-u CPU_COUNT] [-c MAX_CONN] [-s STGE_TYPE]

This script is a bash port of PGTune (https://pgtune.leopard.in.ua).
It produces a postgresql.conf file based on supplied parameters.

  -h                  display this help and exit
  -v PG_VERSION       (optional) PostgreSQL version
                      accepted values: 9.5, 9.6, 10, 11, 12, 13, 14
                      default value: 14
  -t DB_TYPE          (optional) For what type of application is PostgreSQL used
                      accepted values: web, oltp, dw, desktop, mixed
                      default value: web
  -m TOTAL_MEM        (optional) how much memory can PostgreSQL use
                      accepted values: integer with unit ("MB" or "GB") between 1 and 9999 and greater than 512MB
                      default value: this script will try to determine the total memory and exit in case of failure
  -u CPU_COUNT        (optional) number of CPUs, which PostgreSQL can use
                      accepted values: integer between 1 and 9999
                      CPUs = threads per core * cores per socket * sockets
                      default value: this script will try to determine the CPUs count and exit in case of failure
  -c MAX_CONN         (optional) Maximum number of PostgreSQL client connections
                      accepted values: integer between 20 and 9999
                      default value: preset corresponding to db_type
  -s STGE_TYPE        (optional) Type of data storage device used with PostgreSQL
                      accepted values: hdd, ssd, san
                      default value: this script will try to determine the storage type (san not supported) and use hdd
                      value in case of failure.
```

## `test.sh`

The `test.sh` compares some `pgtune.sh` results with pre-generated `postgresql.conf` available in `test_files/`

## Building and running Docker image

The PGTuned image is built on top of the [official PostgreSQL Docker image](https://hub.docker.com/_/postgres). The default tag used is `14`.  
In practice, at container startup `pgtuned.sh` script replaces the default `postgresql.conf` file by a new one created with `pgtune.sh` using supplied options.

### Building pgtuned image

The command below builds the `pgtuned` image using `postgres:14` image **without PostGIS**

`docker build --no-cache . -t pgtuned`

The following command builds the `pgtuned:13` image using `postgres:13` image **without PostGIS**

`docker build --no-cache --build-arg POSTGRES_VERSION=13 . -t pgtuned:13`

The following command builds the `pgtuned:12-2.5` image using `postgres:12` image **with PostGIS 2.5**

`docker build --no-cache --build-arg POSTGRES_VERSION=12 --build-arg POSTGIS_VERSION=2.5 . -t pgtuned:12-2.5`

### Running pgtuned image

`POSTGRES_PASSWORD` environment variable is **compulsory** to use the official PostgreSQL image and therefore the `pgtuned` image.  
All other environment variables of the official PostgreSQL Docker image may also be used.

In addition the following environment variables may be provided to tune PostgreSQL with `pgtune.sh` :
* `DB_TYPE` : If not provided `web` will be used as default DB_TYPE
* `TOTAL_MEM` : If not provided `pgtune.sh` will try to determine the total memory automatically
* `CPU_COUNT` : If not provided `pgtune.sh` will try to determine the cpu count automatically
* `MAX_CONN` : If not provided `200` wil be used as default maximum client connections number
* `STGE_TYPE` : If not provided `pgtune.sh` will try to determine the storage type automatically
* *`PG_VERSION` : Should not be necessary as Docker image `PG_MAJOR` environment variable will be used by default*

Default command line for running the `pgtuned` image with default `pgtune.sh` options :
`docker run -e POSTGRES_PASSWORD=secret --name pgtuned pgtuned`

Command line example for running the `pgtuned` imaage with `2GB` of RAM, `mixed` database type, `4` cpu cores and `ssd` storage :
`docker run -e POSTGRES_PASSWORD=secret -e TOTAL_MEM=2GB -e DB_TYPE=mixed -e CPU_COUNT=4 -e STGE_TYPE=ssd --name pgtuned pgtuned`

You can check PostgreSQL parameter (here `work_mem`) by using such a command once the `pgtuned` is up and running :
```
user@machine:$ docker exec -ti pgtuned psql -U postgres -W -c "show work_mem;"
Password: 
 work_mem 
----------
 1310kB
(1 row)
```

### Using pgtuned with docker-compose

A docker-compose file is provided to illustrate how to use the `pgtuned` image in the context of a docker-compose project.
