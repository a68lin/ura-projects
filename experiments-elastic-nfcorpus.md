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

## Install Elasticsearch (MacOS)

Download Elasticsearch:
```bash
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz
curl https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz.sha512 | shasum -a 512 -c - 
tar -xzf elasticsearch-8.13.2-darwin-x86_64.tar.gz
cd elasticsearch-8.13.2/ 
```

Run Elasticsearch:
```bash
./bin/elasticsearch
```

A password will be genereted as below if the build is successfull, copy the password.
```bash
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  'password'

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  'fingerprint'
```

Open another terminal, cd to the elasticsearch directory. and set two new environment variables:
```bash
export ELASTIC_PASSWORD="your_password" #replace it with your actual passwrod
export ES_HOME="path_to_elasticsearch"
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

Now we need to exit the Elasticsearch server and change its security configuration in config/elasticsearch.yml as below. 
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
This change disables security checks when connecting to Elasticsearch server from clients to help us avoid permission errors. In real-world productions, it is a better idea to keep the security features to protect the server and the data. 

Now restart the Elasticserver again:
```bash
./bin/elasticsearch
```

## Set up a client connection to Elasticsearch:

Now your elasticsearch service should be up and running! Then we will setup an environment for clients to connect to the server. 

We will download some starter code from [elasticsearch-lab](https://github.com/elastic/elasticsearch-labs/raw/main/example-apps/search-tutorial/search-tutorial-starter.zip), this guide will not directly use the code downloaded but instead using interactive python shell to go through every step from indexing to retrieval for NFCorpus. (Throughout this guide, if you exit the python shell, you need to run all the previous code again. Open up new terminals to avoid exiting the shell if you need to do something else)

After downloading, extract the contents from .zip file, cd to the search-tutorial directory, this directory will be our main working directory. 
```bash
cd search-tutorial
```

Create a virtual environment and activate it:
```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install all the required python packages:
```bash
pip install -r requirements.txt
```

Install Elasticsearch and SentenceTransformers python modules, update the requirements.txt to keep it align with your current module versions:
```bash
pip install elasticsearch
pip install sentence-transformers
pip freeze > requirements.txt
```
SentenceTransformers framework is used to generate vectors(embeddings) using pre-trained models, in thie guide, we will be using a model called all-MiniLM-L6-v2. 

Now it's time to open up an interactive Python shell

To setup the connection:
```python
from elasticsearch import Elasticsearch
from pprint import pprint
es = Elasticsearch('http://localhost:9200')
client_info = es.info()
pprint(client_info.body)
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

Now you have setup a client which is connected to your running Elasticsearch service.

## Data Prep

To work with [NFCorpus](https://www.cl.uni-heidelberg.de/statnlpgroup/nfcorpus/), you can download it again in the search-tutorial directory or copy the existing one from previous guides. 

To download NFCorpus, go to the search-tutorial directory: 
```bash
wget https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/nfcorpus.zip -P collections
unzip collections/nfcorpus.zip -d collections
```

We need to convert the corpus from jsonl to json. Additionally, because Field '_id' and 'metedata' are special fields which cannot be added inside a document in Elasticsearch, we need to change these two fields' names. In the Python shell, run the following code:

```python
import json
with open('collections/nfcorpus/corpus.json', 'w') as out:
    with open('collections/nfcorpus/corpus.jsonl', 'r') as f:
        jsonl_data = [json.loads(line.strip()) for line in f]
    for obj in jsonl_data:
        obj["id"] = obj.pop("_id")
        obj["data"] = obj.pop("metadata")
    json.dump(jsonl_data, out, indent=4)
```
Okay, the data are ready now.

## Indexing

We can now "index" these documents:

First, we need to load the pretrained model:
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')
```

In Elasticsearch, in order to create an index:
```python
es.indices.create(index='my_documents', mappings={
            'properties': {
                'embedding': {
                    'type': 'dense_vector',
                    'similarity': 'dot_product',
                }
            }
        })
```

Then we will insert NFCorpus to the index structure in Elasticsearch:

```python
with open('collections/nfcorpus/corpus.json', 'rt') as f:
    documents = json.loads(f.read())
operations = []
for document in documents:
    operations.append({'index': {'_index': 'my_documents'}})
    operations.append({
        **document,
        'embedding': model.encode(document['title']),
    })
es.bulk(operations=operations)
```

A document after inserted:
```bash
{'index': {'_index': 'my_documents', '_id': '6rVe_44BZpkgJxnsW0xf', '_version': 1, 'result': 'created', '_shards': {'total': 2, 'successful': 1, 'failed': 0}, '_seq_no': 0, '_primary_term': 1, 'status': 201}}
```

Now the NFCorpus has been indexed. 

## Retrieval

We can now perform retrieval by Elasticsearch, run following code in the shell:

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
