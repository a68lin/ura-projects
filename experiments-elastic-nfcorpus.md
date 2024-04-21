# Elasticsearch For NFCorpus

This guide contains instructions for running Elasticsearch for NFCorpus.

If you're a Waterloo student traversing the [onboarding path](https://github.com/lintool/guide/blob/master/ura.md) (which [starts here](https://github.com/castorini/anserini/blob/master/docs/start-here.md)),
make sure you've first done the previous step, [A Deeper Dive into Dense and Sparse Representations](https://github.com/castorini/pyserini/blob/master/docs/conceptual-framework2.md).
In general, don't try to rush through this guide by just blindly copying and pasting commands into a shell;
that's what I call [cargo culting](https://en.wikipedia.org/wiki/Cargo_cult_programming).
Instead, really try to understand what's going on.

**Learning outcomes** for this guide, building on previous steps in the onboarding path:

+ Install and run Elasticsearch service and build a client connected to ther server
+ Index documents and perform vetor search in NFCorpus using Elasticsearch

## Install Elasticsearch (MacOS)

What is Elasticsearch? Elasticsearch is a distributed document store. Instead of storing information as rows of columnar data, Elasticsearch stores complex data structures that have been serialized as JSON documents. Elasticsearch also uses inverted index, which is introduced in previous guide, to store documents. In this guide, we will setup an Elasticsearch server locally on your computer, connect to the server and perform vector search on NFCorpus using Elasticsearch. 

Download Elasticsearch on MacOS:
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

In the first time you run Elasticsearch, a password will be genereted. If the build is successfull, you will see a result as below. Copy the password. 
```bash
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  'password'

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  'fingerprint'
```

Open up another terminal, cd to your elasticsearch directory. and set two new environment variables:
```bash
export ELASTIC_PASSWORD="your_password" #replace it with your actual passwrod
export ES_HOME="path_to_elasticsearch"
```

Check that Elasticsearch is running:
```bash
curl --cacert $ES_HOME/config/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200 
```

If Elasticsearch is successfully setup, you should see a result like this:
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

Now we need to exit the Elasticsearch server (by typing Ctrl+C). Go to the config file in in config/elasticsearch.yml and change its security configuration as below. 
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
This change disables all security checks when connecting to Elasticsearch server from clients to help us avoid permission errors. In real-world productions, it is a better idea to keep the security features to protect the server and the data. 

Now restart the Elasticserver again:
```bash
./bin/elasticsearch
```
Now your elasticsearch service should be running and ready to be connected!

## Set up a connection to the Elasticsearch server:

First we need to setup an environment for clients to connect to the server. 

We will download some starter code from: [elasticsearch-lab](https://github.com/elastic/elasticsearch-labs/raw/main/example-apps/search-tutorial/search-tutorial-starter.zip), this guide will not directly use the code downloaded but instead using interactive python shell to go through every step from indexing to retrieval.

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
SentenceTransformers framework is used to generate vectors(embeddings) using pre-trained models. Previous guides show that we need an encoder to encode a text to a vector. Here is a list of [models](https://www.sbert.net/docs/pretrained_models.html#model-overview) that SentenceTransformers supports. In thie guide, we will use a model called `all-MiniLM-L6-v2` as it is recommended in the documentation. 

Now it's time to open up an interactive Python shell. 

To setup the connection:
```python
from elasticsearch import Elasticsearch
from pprint import pprint
es = Elasticsearch('http://localhost:9200')
client_info = es.info()
pprint(client_info.body)
```
To connect to our self-hosted Elasticsearch server, we need to provide the URL to the top-level endpoint of the Elasticsearch service, which is normally http://localhost:9200. `info()` makes a call to the service requesting basic information.

If the connection is valid, you should see a message like this:
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

Now you have setup a client named `es` which is connected to your running Elasticsearch service.

## Data Prep

To work with [NFCorpus](https://www.cl.uni-heidelberg.de/statnlpgroup/nfcorpus/), you can copy the `collections` folder from previous guides to `the search-tutorial` directory or download it again. 

To download NFCorpus, open up a new terminal and go to the search-tutorial directory: 
```bash
wget https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/nfcorpus.zip -P collections
unzip collections/nfcorpus.zip -d collections
```

Since Elasticsearch receives data in json format as input we need to convert the corpus from jsonl to json. Additionally, because Field '_id' and 'metedata' are special fields in Elasticsearch, which cannot be added inside a document, we need to change these two fields' names. 

In the Python shell, run the following code:
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
Okay, the NFCorpus data is ready now.

## Indexing

To perform vector search in Elasticsearch, we need to first create an index in Elasticsearch and inserts documents in NFCorpus into the index. You can think of an index as a collection of documents that is stored in a highly optimized format designed to perform efficient searches. 

First, we need to load the pretrained model as an encoder:
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')
```

In Elasticsearch, in order to create an index:
```python
es.indices.delete(index='nfcorpus_documents', ignore_unavailable=True)
es.indices.create(index='nfcorpus_documents', mappings={
            'properties': {
                'embedding': {
                    'type': 'dense_vector',
                    'similarity': 'dot_product',
                }
            }
        })
```
The first `delete` call is to delete any existing `nfcorpus_documents` index. The `create` call is to initialize an index in Elasticsearch. An index in Elasticsearch contains a mapping which stores all fields that each document has. The `mappings` option in `create` is to explicitly add a vector field in the index, which are latter used in the kNN vector search. In the `embedding` property, we use `dense_vector` and `dot_product`. The type of `dense_vector` is to support fast kNN search, there are also other types of vectors like `sparse_vector`. `dot_product` specifies that dot production will be used to compare vectors. 

Then we will insert NFCorpus documents to the index we just created in Elasticsearch:
```python
with open('collections/nfcorpus/corpus.json', 'rt') as f:
    documents = json.loads(f.read())

operations = []
for document in documents:
    operations.append({'index': {'_index': 'nfcorpus_documents'}})
    operations.append({
        **document,
        'embedding': model.encode(document['title']),
    })

es.bulk(operations=operations)
```
We first load the documents in `corpus.json`. `operations` is a list to store all the operations we want to perform at once using `bulk()`. The first `append` defines an insertion of a documnt to the index 'nfcorpus_documents'. The second `append` is to add a docuement and the generated vector using `all-MiniLM-L6-v2` model we load before. There is an `insert_document` API in Elasticsearch so we can add documents one by one easily but we use `bulk()` instead because performing all insertions at once is more efficient and fast. 

Now the NFCorpus has been indexed. 

## Retrieval

We can now perform vector search using Elasticsearch.

In Elasticsearch, we use k-Nearest Neighbor (kNN) Search to perform vector search. The k-nearest neighbor (kNN) algorithm performs a similarity search on fields of `dense_vector` type.
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
In `search()`, we use `knn` option to perform kNN search. `field` specifies the field in the index used to perform the search and we know that `dense_vector` is in the `embedding` field. `query_vector` is the vector generated from a query, in this demo, we use the same example query in previous guide. `all-MiniLM-L6-v2` model is used again to encode the query. `num_candidates` is the number of candidate documents to consider from each shard. Elasticsearch retrieves this many candidates from each shard, combines them into a single list and then finds the closest "k" to return as results. 'k' specifies the number of results to return. 

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
The top result is `MED-4555` which is the same in previous guides. In the qrels, we can see that `MED-4555` is the only document that is related to our query so the result makes sense.  

And that's the lesson! Before you move on, however, add an entry in the "Reproduction Log" at the bottom of this page, following the same format: use `yyyy-mm-dd`, make sure you're using a commit id that's on the main trunk of pyserini, and use its 7-hexadecimal prefix for the link anchor text.

## Reproduction Log[*](reproducibility.md)

+ Results reproduced by [@a68lin](https://github.com/a68lin) on 2024-04-20 (commit [`7dda9f3`](https://github.com/castorini/pyserini/commit/7dda9f3246d791a52ebfcedb0c9c10ee01d4862d))
