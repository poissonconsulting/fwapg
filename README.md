# fwapg

PostgreSQL tools for working with British Columbia's [Freshwater Atlas](https://www2.gov.bc.ca/gov/content/data/geographic-data-services/topographic-data/freshwater).

## Rationale

BC FWA data can be requested [via WFS](https://openmaps.gov.bc.ca/geo/pub/wfs?service=WFS&version=2.0.0&request=GetFeature&typeName=WHSE_BASEMAPPING.FWA_LAKES_POLY&outputFormat=json&SRSNAME=epsg%3A3005&CQL_FILTER=GNIS_NAME_1=%27Quamichan%20Lake%27) and via WFS wrappers such as:

- [fwapgr (R)](https://github.com/poissonconsulting/fwapgr)
- [fwatlasbc (R)](https://github.com/poissonconsulting/fwatlasbc)
- [bcdata (R)](https://github.com/bcgov/bcdata)
- [bcdata (Python)](https://github.com/smnorris/bcdata)
- [pgsql-ogr-fdw (PostgreSQL extension)](https://github.com/pramsey/pgsql-ogr-fdw)

These work well but using WFS has some specific limitations for Provincal analysis:

- requesting tens of thousands of features is generally stable (with a good internet connection), but for larger tables it can be error prone (there are 4.9M individual stream segments, or 1M for the Fraser system alone)
- non-spatial FWA lookup tables required for some workloads (such as [linking fish observations to waterbodies](https://github.com/smnorris/bcfishobs)) are not published via WFS ([example](https://catalogue.data.gov.bc.ca/dataset/freshwater-atlas-20k-50k-stream-cross-reference-table))
- querying WFS via `CQL_FILTER` expressions works well but the WFS does not specifically support upstream/downstream relationships built into the FWA data

As a zipped geopackage, the FWA is under 7G - by downloading this and loading to Postgres, we can:

- leverage the upstream/downstream materialized paths built into FWA watershed codes by using the Postgres [`ltree` module](https://www.postgresql.org/docs/current/ltree.html) - this enables speedy upstream/downstream queries for answering questions like:
    + [*how many salmon observations are there upstream of a point?*](https://github.com/smnorris/bcfishobs)
    + [*what is the area of a watershed above a stream guage?*](https://github.com/smnorris/bcbasins)
    + [*how many km of potential fish habitat are upstream of a failed culvert?*](https://github.com/smnorris/pscis)
- add additional convenience tables (named streams, optimized watershed groups)
- populate the empty `gradient` column in the streams table for ongoing use
- connect directly to the database for mapping and to run various ad-hoc queries using spatial SQL and tools that support the PostgreSQL / PostGIS ecosystem
- serve FWA data direct from the db as MVT (vector tiles) via [`pg_tileserv`](https://github.com/CrunchyData/pg_tileserv) (example [MVT service](https://www.hillcrestgeo.ca/pg_tileserv))
- serve FWA data features and linear referencing functions via API provided by [`pg_featureserv`](https://github.com/CrunchyData/pg_featureserv) (example [API](https://www.hillcrestgeo.ca/fwapg) and [R client](https://github.com/poissonconsulting/fwapgr/))


## Requirements

- PostgreSQL (requires >= v12)
- PostGIS (tested with >=v3.0.1)
- GDAL (tested with >= v2.4.4)


## Installation

The repository is a collection of sql files and shell scripts - no installation is required.

Download and unzip the [latest release](https://github.com/smnorris/fwapg/releases)

Or get the latest:

    git clone https://github.com/smnorris/fwapg.git
    cd fwapg


## Configuration

The data load scripts require that the [postgres environment variables](https://www.postgresql.org/docs/current/libpq-envars.html) `$PGHOST`, `$PGUSER`,`$PGDATABASE`,`$PGPORT` are set to point at the database you wish to use. For example:

    export PGHOST=localhost
    export PGUSER=postgres
    export PGPORT=5432
    export PGDATABASE=postgis

To provide the script with a password to the database, either [create a password file]( https://www.postgresql.org/docs/current/libpq-pgpass.html) or modify the connection strings in the script.

This document does not cover PostgreSQL setup - this is a detailed topic which depends on hardware and workload. See these links for more:

- [general guide](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [sample setup script for GIS processing on MacOS](https://github.com/bcgov/designatedlands/blob/master/scripts/postgres_mac_setup.sh)


## Setup

Create the target database if it does not already exist, with a command something like this:

    psql -c "CREATE DATABASE $PGDATABASE" postgres

Create the required extensions and schema:

    psql -c "CREATE EXTENSION IF NOT EXISTS postgis"
    psql -c "CREATE EXTENSION IF NOT EXISTS ltree"
    psql -c "CREATE SCHEMA IF NOT EXISTS whse_basemapping"

### Load

Note that all data are licensed under the [Open Government Licence – British Columbia](https://www2.gov.bc.ca/gov/content/data/open-data/open-government-licence-bc)

Download the zipped FWA data file - an extract that includes most FWA tables in a single file - then unzip and run the load script. For example:

    wget https://www.hillcrestgeo.ca/outgoing/public/fwapg/FWA.zip
    unzip FWA.zip
    ./01_load.sh

### Clean / optimize

    ./02_clean.sh

It takes time to build all the indexes but once done you have a Provincial FWA database ready for speedy queries.

## Testing

Extremely basic tests are included for selected functions.
If changing a covered function, run the individual test. For example:

    psql -f tests/test_fwa_upstream.sql

All results should be true.
