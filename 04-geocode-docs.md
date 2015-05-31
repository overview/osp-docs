# Geocode the documents

Next, we can link the documents with a set of US-accredited institutions. These institutions have addresses that can be geocoded, which, by extension, gives us locations for each of the documents that we can pair with an institution. Like with the `document` and `document_text` tables, the institution tables in the existing EBS snapshots are already populated with data. Here's how to build up the data from scratch:

## Insert institution rows

1. SSH into an `osp-server` instance.

1. Drop the institutions table, and the table that links documents with institutions:

```
psql osp -U osp
DROP TABLE institution;
DROP TABLE document_institution;
```

1. Then, close out of psql and recreate the tables:

```
osp inst init_db
osp locations init_db
```

1. Next, insert institution rows from the CSV of US-accredited institutions (this is committed with the repo at `osp/institutions/data/institutions.csv`):

```
osp inst insert_institutions
```

## Geocode the institutions

Now, we can use the MapQuest geocoding API to find coordinates for the institutions.

1. Back on your local machine, open up the encrypted secrets file and make sure a valid `osp_mapquest_key` is entered. The one committed to the repo as of 6/15 should be fine, but, if the geocoding job is being run frequently, it might be necessary to rotate this if one key hits an API limit. If you do change it, before to re-deploy the server with:

```
ansible-playbook deploy.yml
```

Which will update the OSP configuration file on the server.

1. SSH back onto the server, and start the geocoding jobs with:

```
osp inst queue_geocode
```

And, go to the IP address of the server in a browser to see the RQ dashboard that shows the progress of the ~37k jobs. This will only take 30-40 minutes to run, so it generally isn't necessary to break the job across multiple machines.

## Match documents with institutions

Now, we can try to match documents to institutions by comparing the URLs of the institutions with the scrape URLs of the documents.

1. On your local machine, edit `/deploy/vars/ec2.yml` and request 20 workers.

1. Start the worker set with:

```
ansible-playbook provision.yml
```

1. And, kick off the jobs with:

```
osp workers queue_match_doc_inst
```

1. Once it's finished, as always, re-snapshot the extraction volume.
