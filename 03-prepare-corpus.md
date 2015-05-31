# Prepare the corpus

## Insert documents into Postgres

If you're working with one of the existing OSP extraction snapshots, all of the tables in the Postgres database will already be filled in with data from previous extraction runs. But, the next couple sections will walk through the process of building up all of this data from scratch. This might be necessary, for example, if new documents are added to the corpus, and you want to re-run the extraction jobs on the new data.

First, we'll register a row in the database for each of the documents in the corpus:

1. Drop the existing document tables:

```
psql osp -U osp
DROP TABLE document;
DROP TABLE document_text;
```

1. Close out of psql, and re-install the tables:

```
osp corpus init_db
```

1. Then, insert the documents into Postgres with:

```
osp corpus insert_documents
```

Be sure to run this in a tmux session, since it will take a few minutes.

1. Last, we need to snapshot the volume on the `osp-server` instance with the new document rows, which will make it possible to duplicate this data out to the worker set in the next step. Find the volume attached to the instance, right click on it, and hit "Create Snapshot." I've been using the naming convention `osp-extraction-M-DD-YY`.

1. When the snapshot is finished, grab the id of the new snapshot and paste it into the `ec2_osp_snapshot` field in the encrypted `group_vars/all/global.yml` file, like we did before.

## Extract full text

Next, we'll pull the raw text out of the documents and write it into Postgres. This would take 40-50 hours if run on a single machine, so we'll parallelize it across a set of workers.

1. Back on your local machine, open up the `/deploy/vars/ec2.yml` file again, and bump the worker count up to 20:

```
ec2_worker_instance_type: c4.4xlarge
ec2_worker_count: 0
```

1. Then, we need to give Ansible an IP address of the central data server, which will get threaded into configuration files on the workers, which will make them write extraction data into the central data instance (in this case, the document text). Get the IP of the `osp-server` instance, and open up the encrypted secrets file:

```
ansible-vault edit group_vars/all/global.yml
```

1. Paste the IP address in for the `osp_data_host` key.

1. Then, re-run the three playbooks to put up the worker set:

```
ansible-playbook provision.yml
./ec2/ec2.py --refresh-cache
ansible-playbook configure.yml
ansible-playbook deploy.yml
```

1. Now, we can use the OSP command line app to communicate with the workers. To make sure they're all online, run:

```
osp workers ping
```

1. Then, run this command to fire off work orders to the EC2 instances, which will kick off the extraction:

```
osp workers queue_corpus_text
```

1. With 20 c4.4xlarge workers, this will take 3-4 hours to finish. While it runs, monitor the progress with:

```
osp workers status
```

Which, for each worker, shows the number of jobs left in the queue and the number of failed jobs. (For the text extraction job, 3-400 failures per worker is typical - most are malformed or encrypted PDFs.)

1. Once all of the workers have worked their queues down to 0, SSH into the `osp-server` instance and check the row count in the `document_text` table, which should be roughly the number of documents in the corpus.

```
psql osp -U osp
SELECT COUNT(*) FROM document_text;
```

1. At this point, re-snapshot the volume on the extraction database, and update it in `deploy.group_vars/all/global.yml`. And, of course, terminate the worker instances.
