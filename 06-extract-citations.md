# Extract citations

To extract citations, we walk through the full collection of 10M+ MARC records in the Harvard Library Open Metadata collection, and try to find matches in the OSP corpus.

## Index OSP documents in Elasticsearch

First, we need to index the OSP corpus in Elasticsearch.

1. Open the `deploy/vars/elasticsearch.yml` file, and bump up the `es_heap_size` setting to something fairly high, roughly 50% of the RAM of the server instance you plan to start. Eg, if you're using a c4.4xlarge with 30 gigs of ram, set this to ~15g.

1. Spin up a `osp-server` instance and intialize the document mapping:

  ```
  osp corpus_index create
  ```

1. Then, open up a new tmux session and kick off indexing with:

  ```
  osp corpus_index insert
  ```

1. Depending on the instance size, this will take 1-2 hours. If it seems particularly slow, open up the `/etc/elasticsearch/elasticsearcg.yml` file and set these values:

  ```
  bootstrap.mlockall: true
  indices.memory.index_buffer_size: 512mb
  index.store.throttle.type: none
  ```

  And, restart elasticsearch with:

  ```
  sudo service elasticsearch restart
  ```

  And re-start the indexing job.

## Insert HLOM records

Next, we need to read the Harvard MARC files and built out a table in Postgres with a separate row for each individal record, which makes individually addressable by the workers (by default, they're munged together in massive, ~2g MARC files).

1. On the extraction server, drop the existing tables:

  ```
  psql osp -U osp
  DROP TABLE hlom_record;
  DROP TABLE hlom_record_cited;
  DROP TABLE hlom_citation;
  DROP TABLE hlom_node;
  DROP TABLE hlom_edge;
  ```

1. And recreate the tables:

  ```
  osp hlom init_db
  ```

1. And, on the extraction server, (in a tmux session, as usual) run:

  ```
  osp hlom insert_records
  ```

1. This will take 1-2 hours. When it's done, snapshot the volume and update the snapshot id in `/deploy/vars/ec2.yml`.

## Run the citation extraction queries

Now, the fun part - running the citation etraction queries:

1. Spin up 20x workers (c4.4xlarge is a sweet spot, again).

1. Fire off the work orders with:

  ```
  osp workers queue_hlom_query
  ```

  For two reasons, the workers will be slow (and fail a lot of jobs) for the first 10-20 minutes of the run. First, the Elasticsearch JVM has to warm up. And, more important, for the first 30-60 minutes the EBS data will still be streaming storage blocks to the instance as Elasticsearch hits them for the first time.

  Don't worry about the read-timeout failures at the beginning - once everything warms up and everything is humming along, you can just run:

  ```
  osp workers requeue
  ```

  To automatically shovel the failed jobs on all the workers back into the pending queues. (Or, just wait until the end and do it in one fell swoop.)
