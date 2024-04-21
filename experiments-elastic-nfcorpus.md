# Elasticsearch For NFCorpus

This guide contains instructions for running Elasticsearch for NFCorpus.

If you're a Waterloo student traversing the [onboarding path](https://github.com/lintool/guide/blob/master/ura.md) (which [starts here](https://github.com/castorini/anserini/blob/master/docs/start-here.md)),
make sure you've first done the previous step, [(TBA)]().
In general, don't try to rush through this guide by just blindly copying and pasting commands into a shell;
that's what I call [cargo culting](https://en.wikipedia.org/wiki/Cargo_cult_programming).
Instead, really try to understand what's going on.

If you've traversed the onboarding path, by now you've learned (TBA).

In this guide, we're going to go through (TBA)

For this guide, assume that we've already got trained encoders.

**Learning outcomes** for this guide, building on previous steps in the onboarding path:

+ (TBA)

## Install Elasticsearch (Now for MacOS only)

Download Elasticsearch:
```bash
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz
curl https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz.sha512 | shasum -a 512 -c - 
tar -xzf elasticsearch-8.13.2-darwin-x86_64.tar.gz
cd elasticsearch-8.13.2/ 
```

Change security configuration of config/elasticsearch.yml as below: (this changes disable security check when connect to Elasticsearch service)
```bash
# Enable security features
xpack.security.enabled: false

xpack.security.enrollment.enabled: false

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: false
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
```
Run Elasticsearch:
```bash
./bin/elasticsearch
```
Password will be genereted if the build is successfull, copy the password and set it and the current directory as environment variables:
```bash
export ELASTIC_PASSWORD="your_password"
export ES_HOME="your_elasticsearch_directory_location"
```
Check that Elasticsearch is running:
```bash
curl --cacert $ES_HOME/config/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200 
```
You should see a result like this:
```bash
{
  "name" : "Cp8oag6",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "AT69_T_DTp-1qgIJlatQqA",
  "version" : {
    "number" : "8.13.2",
    "build_type" : "tar",
    "build_hash" : "f27399d",
    "build_flavor" : "default",
    "build_date" : "2016-03-30T09:51:41.449Z",
    "build_snapshot" : false,
    "lucene_version" : "9.10.0",
    "minimum_wire_compatibility_version" : "1.2.3",
    "minimum_index_compatibility_version" : "1.2.3"
  },
  "tagline" : "You Know, for Search"
}
```
## Install Python dependency

Create a virtual environment and activate it:
```bash
python3 -m venv .venv
source .venv/bin/activate
```
Install all the required python packages:
```bash
pip install -r requirements.txt
```
Install Elasticsearch and SentenceTransformers python module, update the requirements.txt to keep it align with your current module versions:
```bash
pip install elasticsearch
pip install sentence-transformers
pip freeze > requirements.txt
```
SentenceTransformers framework is used to generate vectors(embeddings) using pretrained models, in thie guide, we will be using a model called all-MiniLM-L6-v2. 

## Data Prep

In this lesson, we'll be working with [NFCorpus](https://www.cl.uni-heidelberg.de/statnlpgroup/nfcorpus/), a full-text learning to rank dataset for medical information retrieval.
The rationale is that the corpus is quite small &mdash; only 3633 documents &mdash; so the latency of CPU-based inference with neural models (i.e., the encoders) is tolerable, i.e., this lesson is doable on a laptop.

Let's first start by fetching the data:

```bash
wget https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/nfcorpus.zip -P collections
unzip collections/nfcorpus.zip -d collections
```

We need to do a bit of data munging to get the corpus into the right format (from jsonl to json). Additionally, because Field '_id' and 'metedata' are metadata fields and cannot be added inside a document in Elasticsearch, we need to change these two fields' names. 
Open a Python shell and run the following code:

```python
import json

with open('collections/fcorpus/corpus.json', 'w') as out:
    with open('collections/nfcorpus/corpus.jsonl', 'r') as f:
        jsonl_data = [json.loads(line.strip()) for line in f]
    for obj in jsonl_data:
        obj["id"] = obj.pop("_id")
        obj["data"] = obj.pop("metadata")
    out.dump(jsonl_data, f, indent=4)
```
Okay, the data are ready now.

## Set up a client connection to Elasticsearch:

Now your Elasticsearch service should be up and running.

To setup the connection, we can have a look at search.py in the current directory:
```python
class Search:
def __init__(self):
    self.model = SentenceTransformer('all-MiniLM-L6-v2')
    self.es = Elasticsearch('http://localhost:9200')
    client_info = self.es.info()
    print('Connected to Elasticsearch!')
    pprint(client_info.body)
```
To initialize a client, run the code in the python shell:
```python
es = Search()
```

You should see a message like this:
```bash
Connected to Elasticsearch!
{'cluster_name': 'ad552eba959043158be67dfc021e91cdc',
 'cluster_uuid': 'ks_HfcCdSf2adeKQEsk9gL',
 'name': 'instance-0000000000',
 'tagline': 'You Know, for Search',
 'version': {'build_date': '2023-11-04T10:04:57.184859352Z',
             'build_flavor': 'default',
             'build_hash': 'b4a62ac808e886ff032700c391f45f1408b2538c',
             'build_snapshot': False,
             'build_type': 'docker',
             'lucene_version': '9.7.0',
             'minimum_index_compatibility_version': '7.0.0',
             'minimum_wire_compatibility_version': '7.17.0',
             'number': '8.13.0'}}
```
Now you have connected to your running Elasticsearch service.

## Indexing

We can now "index" these documents:

```python
es.reindex(collections/nfcorpus/nfcorpus.json)
```

we can see the source code of reindex in search.py:
```python
def reindex(self):
    self.create_index()
    with open('nfcorpus/corpus.json', 'rt') as f:
        documents = json.loads(f.read())
    return self.insert_documents(documents)
```

And source code of insert_documents:
```python
def insert_documents(self, documents):
    operations = []
    for document in documents:
        operations.append({'index': {'_index': 'my_documents'}})
        operations.append({
            **document,
            'embedding': self.get_embedding(document['title']),
        })
    return self.es.bulk(operations=operations)
```
Now the NFCorpus has been indexed. 

## Retrieval

We can now perform retrieval by Elasticsearch, run following code in the shell:

We first load the pretrained model:
```python
model = SentenceTransformer('all-MiniLM-L6-v2')
```
Then we can perform knn search:
```python
results = es.search(
    knn={
        'field': 'embedding',
        'query_vector': model.encode('How to Help Prevent Abdominal Aortic Aneurysms'),
        'num_candidates': 50,
        'k': 10,
    },
)
for num in range(0, results['hits']['total']['value']):
    print(f"{num+1} {results['hits']['hits'][num]['_source']['id']} {results['hits']['hits'][num]['_score']}")
```

We should see result like this:
```bash
1 MED-4555 0.8196324
2 MED-4560 0.7140343
3 MED-1554 0.6715589
4 MED-3180 0.66313064
5 MED-2703 0.65824366
6 MED-1234 0.6541223
7 MED-1643 0.6526732
8 MED-1394 0.6496354
9 MED-3436 0.6441341
10 MED-1512 0.6409336
```

And that's it!

The next lesson will provide [TBA]().
Before you move on, however, add an entry in the "Reproduction Log" at the bottom of this page, following the same format: use `yyyy-mm-dd`, make sure you're using a commit id that's on the main trunk of, and use its 7-hexadecimal prefix for the link anchor text.

## Reproduction Log[*](reproducibility.md)

+ Results reproduced by [@a68lin]() on 2024-04-22 (commit [`TBA`]())
