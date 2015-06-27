# Project setup

## Installing the codebase

1. Install Python 3, if you don't have it. On Mac, homebrew is easiest:

  ```
  ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  brew install python3
  ```

1. Clone the repository for the extraction code:

  ```
  git clone https://github.com/overview/osp.git
  ```

1. Install non-Python dependencies:

  ```
  brew install graphicsmagick boost boost-python
  ```

1. Change down into the repo and create a virtual environment, and install dependencies:

  ```
  pyvenv env
  . env/bin/activate
  pip install -r requirements
  ```

  **Note**: On Debian, if the pip install step throws errors related to SciPy, install these packages with aptitude:

  - gfortran
  - libopenblas-dev
  - liblapack-dev
  - libxml2-dev
  - libgraphicsmagick++1-dev
  - libboost-python-dev

  And, on Ubuntu, it should work to just do:

  - `apt-get -y build-dep python3-scipy`

## Create configuration files

The OSP codebase uses YAML files at `/etc/osp` to set global configuration options. We use config files instead of environment variables because jobs in RQ don't share the same shell as the user, and it's a pain to sync ENV variables.

1. The default configuration file is at `osp/common/config.yml`. Create a file at `/etc/osp/osp.yml` and override all keys with `XXX` as the value. Or, at least, keys that are required by the workflow at any given point. Eg, no need to set the MapQuest API key until you geocode institutions. Mine looks like this, working on OSX:

  ```yaml
  ---

  osp:
    corpus: /path/to/corpus
    bin: /path/to/osp/env/bin
    counts: /path/to/counts.csv

  hlom:
    corpus: /path/to/hlom

  postgres:

    default:
      args:
        user: davidmcclure

  tika:
    server: http://localhost:9998/tika

  mapquest:
    api_key: [KEY]

  network:
    font: /path/to/font.ttf
  ```

1. Last, create `/etc/osp/osp.test.yml`, which can contain selective overrides for the testing environment. To keep the test suite from wiping the development data, point the tests at different instances of Postgres, Elasticsearch, and Redis:

  ```yaml
  ---

  postgres:

    default:
      args:
        database: osp-test

  elasticsearch:
    port: 1337

  redis:
    db: 1
  ```

## Configure Postgres and Elasticsearch

1. Install Postgres and Elasticsearch:

  ```
  brew install postgresql
  brew install elasticsearch
  ```

1. Create databases, enable HSTORE:

  ```
  createdb osp
  psql osp
  > CREATE EXTENSION hstore;

  createdb osp-test
  psql osp-test
  > CREATE EXTENSION hstore;
  ```

1. Start two instances of Elasticsearch - one for development, the other for the test suite. I usually don't daemonize these, and just let them run in a minimized terminal:

  ```
  elasticsearch --config=/usr/local/opt/elasticsearch/config/elasticsearch.yml
  elasticsearch --config=/usr/local/opt/elasticsearch/config/elasticsearch.yml --port=1337
  ```

## Run the tests

Now, if you run `py.test osp` from the repository root, the suite should run and pass.
