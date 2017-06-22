# USGS Dam Removal Database (Snorkle) 

🛀 

### Configuration

*Dev - using postgres user*

1. Set ```SNORKELDB``` environment variable

   1. ```python
      os.environ['SNORKELDB']="postgres://postgres@localhost:5432/snorkel_postgres"
      ```

2. Set configuration in yaml files ```config```, ```credentials```

3. Create blank database ```snorkel_postgres``` 

4. Execute first cell in Snorkle tutorial to apply the schema

   1. ```python
      %load_ext autoreload
      %autoreload 2

      import os
      os.environ['SNORKELDB']="postgres://postgres@localhost:5432/snorkel_postgres"

      from snorkel import SnorkelSession
      session = SnorkelSession()
      ```

5. If input data is in sentences_nlp352, run script ```      https://github.com/UW-Deepdive-Infrastructure/stromatolites_snorkel/remap_sentences.py```

```python
import yaml, psycopg2
from psycopg2.extensions import AsIs

# Connect to Postgres
with open('./credentials', 'r') as credential_yaml:
    credentials = yaml.load(credential_yaml)

with open('./config', 'r') as config_yaml:
    config = yaml.load(config_yaml)

# Connect to Postgres
connection = psycopg2.connect(
    dbname=credentials['postgres']['database'],
    user=credentials['postgres']['user'],
    password=credentials['postgres']['password'],
    host=credentials['postgres']['host'],
    port=credentials['postgres']['port'])
cursor = connection.cursor()

snorkel_connection = psycopg2.connect(
    dbname=credentials['snorkel_postgres']['database'],
    user=credentials['snorkel_postgres']['user'],
    password=credentials['snorkel_postgres']['password'],
    host=credentials['snorkel_postgres']['host'],
    port=credentials['snorkel_postgres']['port'])
snorkel_cursor = snorkel_connection.cursor()

cursor.execute("""
    SELECT DISTINCT(docid) FROM sentences_nlp352;
"""
)

count = 1
for docid in cursor:
    snorkel_cursor.execute("INSERT INTO context (id, type, stable_id) VALUES (nextval('context_id_seq'), 'document', %(stable_id)s)", {"stable_id": docid[0] + "::document:0:0"})
    snorkel_cursor.execute("INSERT INTO document (id, name) VALUES (currval('context_id_seq'), %(docid)s)", {"count" : count, "docid": docid[0]})
    snorkel_connection.commit()
    count += 1

#IMPORT THE SENTENCES DUMP
cursor.execute("""
            SELECT docid, sentid, words, poses, ners, lemmas, dep_paths, dep_parents
            FROM sentences_nlp352 ORDER BY docid, sentid;
            """)
# Need to get document-level offsets for stable_id at the sentence level.
count = 1

doc_char_counts = {}
for sent in cursor:
    parsed_sent = {}
    snorkel_cursor.execute("SELECT id FROM document WHERE name=%(docid)s", {"docid" : sent[0]})
    document_id = snorkel_cursor.fetchone()[0]
    parsed_sent["document_id"] = document_id
    parsed_sent["position"] = sent[1]
    parsed_sent["words"] = sent[2]
    parsed_sent["pos_tags"] = sent[3]
    parsed_sent["ner_tags"] = sent[4]
    parsed_sent["lemmas"] = sent[5]
    parsed_sent["dep_labels"] = sent[6]
    parsed_sent["dep_parents"] = sent[7]
    parsed_sent["text"] = " ".join(word for word in parsed_sent["words"])
    parsed_sent["char_offsets"] = [0 for i in range(len(parsed_sent["words"]))]

    sentence_running_count = 0
    for wordidx in range(len(parsed_sent["words"])):
        parsed_sent["char_offsets"][wordidx] = sentence_running_count
        sentence_running_count += len(parsed_sent["words"][wordidx]) + 1

    # This will probably be off by one...
    if sent[0] in doc_char_counts:
        sentence_start = doc_char_counts[sent[0]] + 1
        doc_char_counts[sent[0]] += sentence_running_count
    else:
        sentence_start = 0
        doc_char_counts[sent[0]] = sentence_running_count

    # keep this running count as the sentence-level offset stable_id
    snorkel_cursor.execute("INSERT INTO context (id, type, stable_id) VALUES (nextval('context_id_seq'), 'sentence', %(stable_id)s)", {"stable_id": sent[0] + "::sentence:%s:%s" % (sentence_start, doc_char_counts[sent[0]])})

    snorkel_connection.commit()
    snorkel_cursor.execute(" \
        INSERT INTO sentence (id, document_id, position, words, pos_tags, ner_tags, lemmas, dep_labels, dep_parents, char_offsets, text) VALUES \
                (currval('context_id_seq'), \
                %(document_id)s, \
                %(position)s, \
                %(words)s, \
                %(pos_tags)s, \
                %(ner_tags)s,  \
                %(lemmas)s, \
                %(dep_labels)s, \
                %(dep_parents)s, \
                %(char_offsets)s, \
                %(text)s);", parsed_sent)
    snorkel_connection.commit()
    count += 1

snorkel_cursor.close()
snorkel_connection.close()
cursor.close()
connection.close()
```

## Running

```sh 
sh run.sh  # in snorkel dir
```

