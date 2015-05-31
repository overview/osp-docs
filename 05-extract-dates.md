# Extract dates

There are three methods for extracting dates from the documents - we look at the date that the document was scraped (least accurate), try to find a "semester" passtern in the document body (eg, "ENG 101" - more accurate), and, in the case of PDF's and MS Office files, try to read a timestamp in the document metadata (most accurate).

1. Assuming you're rebuilding the data from scratch - SSH into the `osp-server` instance and drop the date tables:

```
psql osp -U osp
DROP TABLE document_date_archive_url
DROP TABLE document_date_file_metadata
DROP TABLE document_date_semester
```

1. And then recreate the tables:

```
osp dates init_db
```

1. Then, back on your local machine, start a worker set. (Whenever you start a new worker set, be sure that the `ec2_osp_snapshot` setting in the `globals.yml` file is pointed at the latest extraction snapshot.)

1. And, run the archive URL job:

```
osp workers queue_date_archive_url
```

1. And, when that's finished, the semester pattern job:

```
osp workers queue_date_semester
```

1. And last the file metadata job:

```
osp workers queue_date_file_metadata
```

1. And, last, snapshot the extraction volume on `osp-server`.
