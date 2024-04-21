# Pyserini: Elasticsearch For NFCorpus

This guide contains instructions for running Elasticsearch for NFCorpus.

If you're a Waterloo student traversing the [onboarding path](https://github.com/lintool/guide/blob/master/ura.md) (which [starts here](https://github.com/castorini/anserini/blob/master/docs/start-here.md)),
make sure you've first done the previous step, [A Deeper Dive into Dense and Sparse Representations](https://github.com/castorini/pyserini/blob/master/docs/conceptual-framework2.md).
In general, don't try to rush through this guide by just blindly copying and pasting commands into a shell;
that's what I call [cargo culting](https://en.wikipedia.org/wiki/Cargo_cult_programming).
Instead, really try to understand what's going on.

**Learning outcomes** for this guide, building on previous steps in the onboarding path:

+ Install and run Elasticsearch service and build a client connected to the local Elasticsearch server
+ Index documents in NFCorpus and perform vector searching using Elasticsearch

## Install Elasticsearch

What is Elasticsearch? Elasticsearch provides a distributed document store and searching APIs. Instead of storing information as rows of columnar data, Elasticsearch stores complex data structures that have been serialized as JSON documents. Elasticsearch also uses inverted index, which is introduced in previous guides, to store documents. In this guide, we will set up an Elasticsearch server locally on your computer and perform vector searching on NFCorpus using Elasticsearch. 

Download Elasticsearch and go to the elasticsearch directory:

macOS:
```bash
curl -O https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz
curl https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.13.2-darwin-x86_64.tar.gz.sha512 | shasum -a 512 -c - 
tar -xzf elasticsearch-8.13.2-darwin-x86_64.tar.gz
cd elasticsearch-8.13.2/ 
```

Start Elasticsearch service:
```bash
./bin/elasticsearch
```

The first time you run Elasticsearch, a password will be generated. If the build is successful, you will see a result as below. 
```bash
✅ Elasticsearch security features have been automatically configured!
✅ Authentication is enabled and cluster connections are encrypted.

ℹ️  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  'password'

ℹ️  HTTP CA certificate SHA-256 fingerprint:
  'fingerprint'
```

Launch another terminal, cd to your elasticsearch directory. Copy the password and set two new environment variables:
```bash
export ELASTIC_PASSWORD="your_password" #replace it with your actual password
export ES_HOME="path_to_elasticsearch"
```

Check that Elasticsearch is running normally:
```bash
curl --cacert $ES_HOME/config/certs/http_ca.crt -u elastic:$ELASTIC_PASSWORD https://localhost:9200 
```

If Elasticsearch is successfully set up, you should see a result like this:
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

Now we need to stop the Elasticsearch service (by typing Ctrl+C). Go to the config file in config/elasticsearch.yml and change the security configuration as below. 
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
This disables all security checks when connecting to the Elasticsearch server which helps us to avoid permission errors. In real-world productions, it's better to enable the security features to protect the server and the data. 

Now restart the Elasticsearch service again:
```bash
./bin/elasticsearch
```
Now your elasticsearch service should be running and ready to be connected!

## Set up a connection to the Elasticsearch service:

First, we need to set up an environment for clients to connect to the server. 

We will download some starter code from: [elasticsearch-lab](https://github.com/elastic/elasticsearch-labs/raw/main/example-apps/search-tutorial/search-tutorial-starter.zip). Note that this guide will not directly use the code downloaded but instead using an interactive python shell to go through every step from indexing to retrieval.

Extract the contents from the downloaded .zip file, cd to the search-tutorial directory, this directory will be our main working directory. 
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

Install Elasticsearch and SentenceTransformers python modules and update the requirements.txt to keep it aligned with your current module versions:
```bash
pip install elasticsearch
pip install sentence-transformers
pip freeze > requirements.txt
```
SentenceTransformers framework is used to generate vectors(embeddings) using pre-trained models. Previous guides show that we need an encoder to generate vectors based on texts. Here is a list of [models](https://www.sbert.net/docs/pretrained_models.html#model-overview) that SentenceTransformers supports. In this guide, we will use a model called `all-MiniLM-L6-v2` as it is recommended in the documentation. 

Now it's time to launch an interactive Python shell. 

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

Now you have set up a client named `es` which is connected to your running Elasticsearch service.

## Data Preparation

To work with [NFCorpus](https://www.cl.uni-heidelberg.de/statnlpgroup/nfcorpus/), you can copy the `collections` folder from previous guides to the `search-tutorial` directory or download it again. 

To download NFCorpus, launch a new terminal and go to the search-tutorial directory: 
```bash
wget https://public.ukp.informatik.tu-darmstadt.de/thakur/BEIR/datasets/nfcorpus.zip -P collections
unzip collections/nfcorpus.zip -d collections
```

Since Elasticsearch receives data in json format we need to convert the corpus from jsonl to json. Additionally, because '_id' and 'metedata' are special fields in Elasticsearch, which cannot be added inside a document, we need to change these two fields' names. 

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

To perform vector searching in Elasticsearch, we need to first create an index in Elasticsearch and insert all documents in NFCorpus to the index. You can think of an index as a collection of documents that is stored in a highly optimized format designed to perform efficient searches. 

First, we need to load the pre-trained model as an encoder:
```python
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')
```

In Elasticsearch, to create an index:
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
The first `delete` call is to delete any existing `nfcorpus_documents` index. The `create` call is to initialize an index in Elasticsearch. An index in Elasticsearch contains a mapping which stores all fields that each document has. The `mappings` option in `create` is to explicitly add a vector field in the index, which is latter used in the kNN vector searching. In the `embedding` property, we use `dense_vector` and `dot_product`. The `dense_vector` vector type is to support fast kNN search, there are also other types of vectors like `sparse_vector`. `dot_product` specifies that dot products of document vectors and query vectors will be used for comparison. 

Then we will insert NFCorpus documents to the index we just created:
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
We first load the documents in `corpus.json`. `operations` is a list storing all the operations we want to perform at once using `bulk()`. The first `append` defines an insertion of a document to the index 'nfcorpus_documents'. The second `append` is to add a document and the generated vector using `all-MiniLM-L6-v2` model we loaded before. There is an `insert_document` API in Elasticsearch so we can add documents one by one easily but we use `bulk()` instead because performing all insertions at once is more efficient and faster. 

Now the NFCorpus has been indexed. 

## Retrieval

We can now perform vector search using Elasticsearch.

In Elasticsearch, we use k-Nearest Neighbor (kNN) Search to perform vector searching. The k-nearest neighbor (kNN) algorithm performs a similarity search on fields of `dense_vector` type.
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
In `search()`, we use `knn` option to perform kNN search. `field` specifies the field in the index used to perform the search and `embedding` is used because `dense_vector` is in it. `query_vector` is the vector generated from a query. The query `'How to Help Prevent Abdominal Aortic Aneurysms'` is used in previous guides as well. `all-MiniLM-L6-v2` model is used again to encode the query. `num_candidates` is the number of candidate documents to consider from each shard. Elasticsearch retrieves this many candidates from each shard, combines them into a single list and then finds the closest 'k' to return as results. 'k' specifies the number of results to return. 

We should see a result like this:
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
The top result is `MED-4555` which corresponds to the results from previous guides. In NFCorpus qrels `test.tsv`, we can see that `MED-4555` is the only document that is related to our query `PLAIN-3074` so the result makes sense.  

And that's the lesson! Before you move on, however, add an entry in the "Reproduction Log" at the bottom of this page, following the same format: use `yyyy-mm-dd`, make sure you're using a commit id that's on the main trunk of pyserini, and use its 7-hexadecimal prefix for the link anchor text.

## Reproduction Log[*](reproducibility.md)

+ Results reproduced by [@a68lin](https://github.com/a68lin) on 2024-04-20 (commit [`7dda9f3`](https://github.com/castorini/pyserini/commit/7dda9f3246d791a52ebfcedb0c9c10ee01d4862d))
