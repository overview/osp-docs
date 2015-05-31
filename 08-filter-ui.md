# Search / Filter UI

Last, we'll look at how to prepare and deploy the main text search and filtering interface.

## Generate term counts

In order to scrub out invalid citations from the interface (eg, junk MARC records from the Harvard that have common words for both the title and author - things like "Health, World Bank"), we need to build out a list of term frequencies for the OSP corpus. This will make it possible for the code to exclude texts that don't contain words that fall above a certain threshold of "rarity" or "focus."

1. Spin up a large `osp-server` instance, ~c4.4xlarge.

1. In a tmux session, run:

```
osp corpus_csv term_counts counts.csv
```

1. This will take 5-6 hours. When it finishes, open the OSP configuration file (`/etc/osp/osp.yml`) and set the value of `osp.counts` to the path of the new `counts.csv` file.

## Denornalize location metadata

To make the location filtering queries run quickly, we need to denormalize state and institution metadata directly onto the `hlom_citation` rows, which makes it possible to avoid costly joins at query time.

1. Open up an IPython shell, and run:

```
from osp.citations.hlom.models.citation import HLOM_Citation
HLOM_Citation.index_institutions()
```

## Prepare the cropped HLOM records table

Next, another performance trick - of the ~10M rows in the `hlom_record` table, only about 500k are actually cited in the syllabi. But, when running the filtering queries, Postgres still has to contend with those extra 9.5 million rows that will never actually show up in the results, which slows things down. To fix this, we create a second version of the records table called `hlom_record_cited`, and copy in just the rows that get cited.

1. From an IPython shell, and run:

```
from osp.citations.hlom.models.record_cited import HLOM_Record_Cited
HLOM_Record_Cited.copy_records()
```

1. Last, we need to denormalize unfiltered ranks and citation counts onto the copied rows - this makes it possible to do regular searches on the title and author fields directly out of Elasticsearch, instead of having to merge the results with ranking data from Postgres:

```
HLOM_Record_Cited.rank()
```

## Index Elasticsearch

Last, we need to index documents, texts, and institutions in Elasticsearch.

1. HLOM records:

```
from osp.citations.hlom.models.record_cited import HLOM_Record_Cited
HLOM_Record_Cited.es_reset()
HLOM_Record_Cited.es_insert()
```

1. Institutions:

```
from osp.institutions.models.institution import Institution
Institution.es_reset()
Institution.es_insert()
```

1. Syllabi:

```
from osp.corpus.models.text import Document_Text
Document_Text.es_reset()
Document_Text.es_insert()
```

1. At this point, snapshot the extraction volume and terminate the instance.

## Deploy the UI

Last but not least, we can deploy the actua UI.

1. Clone the `osp-filter` repo. Like with the `osp` and `osp-network` codebases, use `ansible-vault` to edit the `deploy/group_vars/all/global.yml` file, and paste in the ID of the most recent snapshot with the prepared Postgres and Elasticsearch data directories.

1. And, deploy the code with the regular sequence of playbooks:

```
ansible-playbook provision.yml
./ec2/ec2.py --refresh-cache
ansible-playbook configure.yml
ansible-playbook deploy.yml
```
