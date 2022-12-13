# flask-elastic-nlp
![](docs/img/similar_image.gif)

## Important initial thoughts
This code is a proof-of-concept to showcase the simplicity of similarity image search using NLP and kNN search 
implementation in Elastic stack. The code is not meant to be deployed in the production environment but used as 
a source of inspiration.

## Requirements
### Elasticsearch version
v8.5.2+

### Required models
To successfully implement a similarity image search, you need to import 1 NLP model.
- [sentence-transformers/clip-ViT-B-32-multilingual-v1](https://huggingface.co/sentence-transformers/clip-ViT-B-32-multilingual-v1)

### Elasticsearch resources
In this example, we have only one model imported, and you need smaller nodes of 8GB of memory.

However, if you intend to run multiple NLP models in parallel, you will need more memory to assign to Elasticsearch nodes because models are loaded into memory. 

If your computer does not have enough memory, then you can configure nodes with less memory and always run only 1 or 2 models 
at the same time, depending on how much memory you have available.

To change the value of the memory in the docker-compose, go to `es-docker/.env` file and change `MEM_LIMIT`.

### Python environment
**Python v3.9+**

Make sure, you can access and use Python.

## How to
Before starting the Flask application and using similarity search on your images, you must set up an Elasticsearch cluster with data (indices) and NLP models.

By Data, I mean Elasticsearch index (or more), which contains a document per image with its image embedding. The image embedding is the vector describing the image features generated by an OpenAI model. 

To do so, follow the instructions below.

### 0. Setup Python env
We must set up a Python environment to use scripts for image embeddings and the Flask app. 
```bash
$ git clone https://github.com/radoondas/flask-elastic-image-search.git
$ cd flask-elastic-image-search
$ python3 -m venv .venv
$ source .venv/bin/activate
$ pip install -r requirements.txt
```

### 1. Elasticsearch cluster
You can use the docker-compose bundled in the repository, your cluster, or the ESS cloud with [free trial period](https://cloud.elastic.co/registration).
To run the Elasticsearch cluster locally, use the following docker-compose example.
```bash
$ cd es-docker
$ docker-compose up -d
```
Check if the cluster is running using Kibana or `curl`.

Once the cluster is up and running, let's get the CA certificate out from the Elasticsearch cluster so we can use it in the rest of the setup.
```bash
$ # still in the folder "es-docker"
$ docker cp image-search-85-es01-1://usr/share/elasticsearch/config/certs/ca/ca.crt ../app/conf/ca.crt
```

### 2. Load NLP models 
Let's load the NLP model into the application. You will use the `eland` client to load the models. For more details, follow the [documentation](https://www.elastic.co/guide/en/elasticsearch/client/eland/current/index.html).

Go back in to the main project directory and import the model using Eland docker image.
```bash
# wait until each model is loaded and started. If you do not have enough memory, you will see errors sometimes confusing
$ eland_import_hub_model --url https://elastic:changeme@127.0.0.1:9200 --hub-model-id sentence-transformers/clip-ViT-B-32-multilingual-v1 --task-type text_embedding --start --ca-certs app/conf/ca.crt
```
Example output:
```bash
eland_import_hub_model --url https://elastic:changeme@127.0.0.1:9200 --hub-model-id sentence-transformers/clip-ViT-B-32-multilingual-v1 --task-type text_embedding --start --ca-certs app/conf/ca.crt
2022-12-12 13:40:52,308 INFO : Establishing connection to Elasticsearch
2022-12-12 13:40:52,327 INFO : Connected to cluster named 'image-search-8.5.3' (version: 8.5.3)
2022-12-12 13:40:52,328 INFO : Loading HuggingFace transformer tokenizer and model 'sentence-transformers/clip-ViT-B-32-multilingual-v1'
/Users/rado/pmm_workspace/ml-nlp-demo/flask-elastic-image-search/.venv/lib/python3.9/site-packages/transformers/models/distilbert/modeling_distilbert.py:217: TracerWarning: torch.tensor results are registered as constants in the trace. You can safely ignore this warning if you use this function to create tensors out of constant variables that would be the same every time you call this function. In any other case, this might cause the trace to be incorrect.
  mask, torch.tensor(torch.finfo(scores.dtype).min)
2022-12-12 13:41:03,032 INFO : Creating model with id 'sentence-transformers__clip-vit-b-32-multilingual-v1'
2022-12-12 13:41:03,050 INFO : Uploading model definition
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 129/129 [00:42<00:00,  3.01 parts/s]
2022-12-12 13:41:45,902 INFO : Uploading model vocabulary
2022-12-12 13:41:46,120 INFO : Starting model deployment
2022-12-12 13:41:52,825 INFO : Model successfully imported with id 'sentence-transformers__clip-vit-b-32-multilingual-v1'
```
You can verify that all models are up and running in Kibana: `Machine Learning -> Trained models`

![](docs/img/models.png)

If you see on the screen that some models are missing and you see a message `ML job and trained model synchronization required`, go ahead and click the link to synchronize models.

![](docs/img/model-sync.png)

### 3. Generate image embeddings
Your next step is to generate the image embeddings from your photos. These embeddings will be used for kNN (vector) search in Elasticsearch.

**Put all your photos in to the folder `app/static/images`.** It is up to you how you do this. 

**Notes**:
- you can use sub-folders to maintain sane structure for your images
- only jp(e)g file types were tested
- you need to have hundreds of photos to get best results. If you have only a dozen of them, then vector search in the space you create is minimal, and distances between images (vectors) are very similar.

```bash
$ cd image_embeddings
$ python3 create-image-embeddings.py --es_host='https://127.0.0.1:9200' \
  --es_user='elastic' --es_password='changeme' \
  --ca_certs='../app/conf/ca.crt'
```

After the script finishes, you can check if the index `my-image-embeddings` exists and has documents. Use Kibana Dev Tools to check.
```
GET _cat/indices/my-image-embeddings?v
```
```
health status index               uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   my-image-embeddings vfA3wOheT1C79R-PceDyXg   1   1       1222            0     24.4mb         12.2mb
```
 
### 4. Run flask app
It is now time to spin up the Flask application and search your images using natural language.

Make sure that Python environment is set and all requirements are installed as described above.
```bash
# In the main directory 
# !!! configure file `.env` with values pointing to your Elasticsearch cluster
$ flask run --port=5001
# Access URL `127.0.0.1:5001`
```

## How to run Flask app in Docker 
To run the application in a Docker container, we need to build it and then run the Docker image with the Flask application.
```bash
$ # just make sure you are in the main project directory 
$ cd flask-elastic-nlp
````

### Build the image
In order to be able to run the application in the Docker environment, we need to build the image locally. Because this 
is a Python application with dependencies, the build of the image might take longer. All the requirements are installed.   
```bash
$ docker build . --tag flask-elastic-image-search:0.0.1
```
Once, the build is complete, we can verify if the image is available.
```bash
$ docker images | grep flask-elastic-image-search
```

### Run the image
To run the application, we need to run the Docker image. 

#### Using local (Docker) Elastic stack
From the CLI we need to run the image using the `docker run` command.
```bash
$ docker run --network image-search-85_default -p 5001:5001  \
  -e ES_HOST='https://es01:9200' -e ES_USER='elastic' \
  -e ES_PWD='changeme' flask-elastic-image-search:0.0.1
```
Notes:
- Option `--network image-search-85_default` is important for the application to connect to the Elastic cluster in your 
  Docker environment
- Variable `ES_HOST='https://es01:9200'` is using Docker alias in the network. If you used the Docker compose file as is, 
  you do not need to change the url for Elasticsearh

#### Using external (e.g. ESS) Elastic stack
From the CLI we need to run the image using the `docker run` command. By external Elastic stack we mean non-dockerized, 
or self-managed, or ESS cloud deployment.
```bash
$ docker run -p 5001:5001  \
  -e ES_HOST='https://URL:PORT' -e ES_USER='elastic' \
  -e ES_PWD='changeme' flask-elastic-image-search:0.0.1
```
Notes: 
- Change `ES_HOST='https://URL:PORT'` to your ELasticsearch URL+PORT

### Access the application
The application is now up and running and is accessible on `http://127.0.0.1:5001` 

#### Important note
When the application starts up, it needs to download a model from the Internet, which will take some time. The total time the application start takes depends on your network connection. 

You might see in the terminal similar output.
```bash
$ docker run --network image-search-85_default -p 5001:5001  \
  -e ES_HOST='https://es01:9200' -e ES_USER='elastic' \
  -e ES_PWD='changeme' flask-elastic-image-search:0.0.1
 * Serving Flask app 'flask-elastic-nlp.py' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
Downloading: 100%|██████████| 690/690 [00:00<00:00, 743kB/s]
Downloading: 100%|██████████| 4.03k/4.03k [00:00<00:00, 3.83MB/s]
Downloading: 100%|██████████| 525k/525k [00:00<00:00, 669kB/s]  
Downloading: 100%|██████████| 316/316 [00:00<00:00, 283kB/s]
Downloading: 100%|██████████| 605M/605M [03:20<00:00, 3.02MB/s] 
Downloading: 100%|██████████| 389/389 [00:00<00:00, 344kB/s]
Downloading: 100%|██████████| 604/604 [00:00<00:00, 526kB/s]
Downloading: 100%|██████████| 961k/961k [00:01<00:00, 716kB/s]  
Downloading: 100%|██████████| 1.88k/1.88k [00:00<00:00, 1.67MB/s]
Downloading: 100%|██████████| 116/116 [00:00<00:00, 123kB/s]
Downloading: 100%|██████████| 122/122 [00:00<00:00, 106kB/s]
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5001
 * Running on http://192.168.48.2:5001
Press CTRL+C to quit
```

![](docs/img/app.png)
![](docs/img/endless-route.png)