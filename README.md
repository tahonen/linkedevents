# Linked events

[![Build status](https://travis-ci.org/City-of-Helsinki/linkedevents.svg)](https://travis-ci.org/City-of-Helsinki/linkedevents)
[![codecov](https://codecov.io/gh/City-of-Helsinki/linkedevents/branch/master/graph/badge.svg)](https://codecov.io/gh/City-of-Helsinki/linkedevents)
[![Requirements](https://requires.io/github/City-of-Helsinki/linkedevents/requirements.svg?branch=master)](https://requires.io/github/City-of-Helsinki/linkedevents/requirements/?branch=master)
[![Gitter](https://img.shields.io/gitter/room/City-of-Helsinki/heldev.svg?maxAge=2592000)](https://gitter.im/City-of-Helsinki/
heldev)

Linked Events is event information disseminator and aggregator (events as in things where people participate). At its core is
an implementation of the Linked Events REST API. This API allows interested parties to retrieve information about events and,
if properly authorized, publish new events. Linked Events also contains a framework for retrieving events from legacy
sources in all manner of formats (this will require coding, of course). In short, Linked Events allows you to set up an
event publication hub.

Linked Events for was originally written for the City of Helsinki. [The Linked Events API for the Helsinki capital region](http://api.hel.fi/linkedevents/) contains data from the Helsinki City Tourist & Convention Bureau, the City of Helsinki Cultural Office and the Helmet metropolitan area public libraries. Viewing the API should give a reasonable view for kinds of information Linked Events is targeted for.

This README is targeted for someone wanting to quickly set up Linked Events for development or test drive.

Requirements for running Linked Events
--------------------------------------
* Python 3.6
* Elasticsearch (for /search endpoint, non-fuzzy search works fine without it)
* Docker and docker-compose (if are going to be running in Docker)

Setting up on local machine
---------------------------
If you are going to use Docker, no local setup is necessary. Python dependencies, databases etc. will be
inside containers. You only need step one here, ie. clone the repository

1. Clone this repository to a directory of your choosing (well, you've probably done this already :)

2. These instructions assume an $INSTALL_BASE, like so:
```bash
INSTALL_BASE=$HOME/linkedevents
```

3. Prepare Python 3.x virtualenv using your favorite tools and activate it. Plain virtualenv is like so:
```bash
virtualenv -p python3 $INSTALL_BASE/venv
source $INSTALL_BASE/venv/bin/activate
```
Install required Python packages into the virtualenv
```bash
cd $INSTALL_BASE/linkedevents
pip install -r requirements.txt
```

4. Create the database, like so: (we have only tested on PostgreSQL)
```bash
cd $INSTALL_BASE/linkedevents
sudo -u postgres createuser -R -S linkedevents
# Following is for US locale, we are not certain whether Linkedevents
# behaves differently depending on DB collation & ctype
#sudo -u postgres createdb -Olinkedevents linkedevents
# This is is for Finnish locale
sudo -u postgres createdb -Olinkedevents -Ttemplate0 -lfi_FI.UTF-8 linkedevents
# Create extensions in the database
sudo -u postgres psql linkedevents -c "CREATE EXTENSION postgis;"
sudo -u postgres psql linkedevents -c "CREATE EXTENSION hstore;"
```

Configuring Linked Events
-------------------------
This applies to both local install and running in Docker.

All configuration is handled through environment variables. For development, `config_dev.env`
file is the simplest to configure these. Begin by renaming `config_dev.env.example` to `config_dev.env`.

The example file is quite profusely commented and you probably want to skim through it. If you just want to see what Linked Events does, you do not need to change anything in the example file. The setting you are most likely to need to change is `DATABASE_URL` if your database is named differently, or needs username and password. The next most important setting is  `LANGUAGES`. It affects which language fields are present in the database and search indexes. Ie. it should be changed
carefully.

The next things you are likely to need are `TOKEN_AUTH_ACCEPTED_AUDIENCE` and `TOKEN_AUTH_SHARED_SECRET`. These are needed if you wish to make authenticated requests to the API using a signed token. This will need a configured Tunnistamo-instance (https://github.com/City-of-Helsinki/tunnistamo). There is also the option of making requests using a traditional API key, but this is managed through administration interface along with all other permissions. If you are developing Linked Events, the administrator of the environment can probably provide you with the TOKEN_AUTH_* variables, if they are needed.

Initializing the database
-------------------------
Linked Events stores everything except event images in the database. Thus it will need to be initialized before it can serve any requests.

### Prerequisites

`DATABASE_URL` and `LANGUAGES` must be correctly configured (see "Configuring Linked Events")

If you are running with Docker, you will need to bring up the containers:
`docker-compose up`
Also if you wish to use the examples verbatim, set `DOCKER_CMD` to the name of the command that can be used to execute in the backend container. For the included docker-compose.yml this would be:
`DOCKER_CMD=docker exec linkedevents-backend`

For local setup, make sure your database engine is running and the database has been created (see "Setting on local machine")

### Mandatory initialization steps

If you wish to install Linkedevents without any Helsinki specific data (an empty database), and instead customize everything for your own city, these are the only steps you need.

```
# This fills the database with a basic skeleton
$DOCKER_EXEC python manage.py migrate
# This adds language fields based on settings.LANGUAGES (which may be missing in external dependencies)
$DOCKER_EXEC python manage.py sync_translation_fields
```

### Optional initialization setup

You can optionally add several datasets to your Linked Events instance:
* location data from Helsinki metropolitan region
* event data from the same
* YSO (General Finnish Ontology) keywords
* Helsinki audience and topic categorizations

The first three are very good for testing your instance. The latter two (interlaced sets ;) are also needed if you wish to try the Helsinki UI (https://linkedevents.hel.fi) available from https://github.com/City-of-Helsinki/linkedevents-ui. The UI is somewhat Helsinki specific, but adopting it to your own uses should not be too difficult.

```bash
cd $INSTALL_BASE/linkedevents
# Import general Finnish ontology (used by Helsinki UI and Helsinki events)
$DOCKER_EXEC python manage.py event_import yso --all
# Add keyword set to display in the UI event audience selection
$DOCKER_EXEC python manage.py add_helsinki_audience
# Add keyword set to display in the UI main category selection
$DOCKER_EXEC python manage.py add_helfi_topics
# Import places from Helsinki metropolitan region service registry (used by events from following sources)
$DOCKER_EXEC python manage.py event_import tprek --places
# Import events from Helsinki metropolitan region libraries
$DOCKER_EXEC python manage.py event_import helmet --events
# Import events from Espoo
$DOCKER_EXEC python manage.py event_import espoo --events
# Import City of Helsinki hierarchical organization for UI user rights management
$DOCKER_EXEC python manage.py import_organizations https://api.hel.fi/paatos/v1/organization/ -s helsinki:ahjo
# install API frontend templates:
$DOCKER_EXEC python manage.py install_templates helevents
```

The last command installs the `helevents/templates/rest_framework/api.html` template,
which contains Helsinki event data summary and license. You may customize the template
for your favorite city by creating `your_favorite_city/templates/rest_framework/api.html`.
For further erudition, take a look at the DRF documentation on [customizing the browsable API](http://www.django-rest-framework.org/topics/browsable-api/#customizing)

After this, everything but search endpoint (/search) is working. See [search](#search)

Production notes
----------------

Development installation above will give you quite a serviceable production installation for lightish usage. You can serve out the application using your favorite WSGI-capable application server. The WSGI-entrypoint for Linked Events is ```linkedevents.wsgi``` or in file ```linkedevents/wsgi.py```. Former is used by gunicorn, latter by uwsgi. The callable is ```application```.

You will also need to serve out ```static``` and ```media``` folders at ```/static``` and ```/media``` in your URL space.

Running tests
------------
Tests must be run using an user who can create (and drop) databases and write the directories
your linkedevents installation resides in. Also the template database must include Postgis and
HSTORE-extensions. If you are developing, you probably want to give those
permissions to the database user configured in your development instance. Like so:

```bash
# Change this if you have different DB user
DATABASE_USER=linkedevents
# Most likely you have a postgres system user that can log into postgres as DB postgres user
sudo -u postgres psql << EOF
ALTER USER "$DATABASE_USER" CREATEDB;
\c template1
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS hstore;
EOF

```

Afterwards you can run the tests:
```bash
cd $INSTALL_BASE/linkedevents
py.test events
```

Note that search tests will fail unless you configure [search](#search)

Requirements
------------

Linked Events uses two files for requirements. The workflow is as follows.

`requirements.txt` is not edited manually, but is generated
with `pip-compile`.

`requirements.txt` always contains fully tested, pinned versions
of the requirements. `requirements.in` contains the primary, unpinned
requirements of the project without their dependencies.

In production, deployments should always use `requirements.txt`
and the versions pinned therein. In development, new virtualenvs
and development environments should also be initialised using
`requirements.txt`. `pip-sync` will synchronize the active
virtualenv to match exactly the packages in `requirements.txt`.

In development and testing, to update to the latest versions
of requirements, use the command `pip-compile`. You can
use [requires.io](https://requires.io) to monitor the
pinned versions for updates.

To remove a dependency, remove it from `requirements.in`,
run `pip-compile` and then `pip-sync`. If everything works
as expected, commit the changes.

Search
------
Linkedevents uses Elasticsearch for generating results on the /search-endpoint. If you wish to use that functionality, proceed like so:

1. Install elasticsearch

    We've only tested using the rather ancient 1.7 version. Version 5.x will certainly not work as the `django-haystack`-library does not support it. If you are using Ubuntu 16.04, 1.7 will be available in the official repository.

2. (For Finnish support) Install elasticsearch-analyzer-voikko, libvoikko and needed dictionaries

    `/usr/share/elasticsearch/bin/plugin -i fi.evident.elasticsearch/elasticsearch-analysis-voikko/0.4.0`
    This specific command is for Debian derivatives. The path to `plugin` command might be different on yours. Note that version 0.4.0 is the one compatible with Elasticsearch 1.7

    Installing libvoikko:
    `apt-get install libvoikko1`

    Installing the dictionaries (v5 dictionaries are needed for libvoikko version included in Ubuntu 16.04):

    ```
    wget -P $INSTALL_BASE http://www.puimula.org/htp/testing/voikko-snapshot-v5/dict-morpho.zip
    unzip $INSTALL_BASE/dict-morpho.zip -d /etc/voikko
    ```

3. Configure the thing

    Set the `ELASTICSEARCH_URL` environment variable (or variable in `config_dev.toml`, if you are running in development mode) to your elasticsearch instance. The default value is `http://localhost:9200/`.

    Haystack configuration for all Linkedevents languages happens automatically if `ELASTICSEARCH_URL` is set, but you may customize it manually using `local_settings.py` if you know Haystack and wish to do so.

4. Rebuild the search indexes

   `python manage.py rebuild_index`

   You should now have a working /search endpoint, give or take a few.

Event extensions
----------------

It is possible to extend event data and API without touching `events` application by implementing separate extension applications. These extensions will be wired under field `extension_<extension idenfier>` in event API. If not auto enabled (see 6. below), extensions can be enabled per request using query param `extensions` with comma separated identifiers as values, or `all` for enabling all the extensions.

To implement an extension:

1) Create a new Django application, preferably named `extension_<unique identifier for the extension>`.

2) If you need to add new data for events, implement that using model(s) in the extension application.

3) Inherit `events.extensions.EventExtension` and implement needed attributes and methods. See [extensions.py](events/extensions.py) for details.

4) Add `event_extension: <your EventExtension subclass>` attribute to the extension applications's `AppConfig`.

5) Make the extension available by adding the extension application to `INSTALLED_APPS`.

6) If you want to force the extension to be enabled on every request, add the extension's identifier to `AUTO_ENABLED_EXTENSIONS` in Django settings. 

For an example extension implementation, see [course extension](extension_course).