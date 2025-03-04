```python
#!pip install minio
```

# Minio

## Use case

Minio est un gestionnaire distribué Open Source de stockage d'objets hautes performances. Minio permet donc de gérer des vidéos, des images, des documents pdfs par exemple. Minio est compatible et interfacable avec le système d'AWS de buckets. 

## Installation

avec docker

```
docker run \
   -p 9000:9000 \
   -p 9001:9001 \
   -e "MINIO_ROOT_USER=root" \
   -e "MINIO_ROOT_PASSWORD=rootpassword" \
   quay.io/minio/minio server /data --console-address ":9001"

```

avec `docker-compose.yml`


```
services:
  minio:
    container_name: minio
    image: quay.io/minio/minio
    ports:
        - 9000:9000
    environment:
      MINIO_ROOT_USER: root
      MINIO_ROOT_PASSWORD: rootpassword
    command: server /data
```

## Utilisation 

Comme expliqué plus haut, Minio permet de stocker et de gérer des médias. On peut 


```python
from minio import Minio
```

Pour se connecter et créer un client Minio il suffit de le configurer.


```python
client = Minio("localhost:9000", "root", "rootpassword", secure=False)
```

## Buckets

Les buckets sont la structure de base de stockage. Ils sont la notion la plus grossière, c'est le dossier parent. Les buckets permettent de stocker tout un ensemble de hiérarchie de fichiers.

### Créer et supprimer des buckets

Pour créer un bucket


```python
client.make_bucket("my-bucket")
```


```python
client.list_buckets()
```




    [Bucket('my-bucket'),
     Bucket('my-bucket1'),
     Bucket('my-bucket10'),
     Bucket('my-bucket2'),
     Bucket('my-bucket3'),
     Bucket('my-bucket4'),
     Bucket('my-bucket5'),
     Bucket('my-bucket6'),
     Bucket('my-bucket7'),
     Bucket('my-bucket8'),
     Bucket('my-bucket9')]



On ne peut pas créer deux buckets avec le même nom.


```python
client.make_bucket("my-bucket")
```


    ---------------------------------------------------------------------------

    S3Error                                   Traceback (most recent call last)

    <ipython-input-92-5082a41625e0> in <module>
    ----> 1 client.make_bucket("my-bucket")
    

    ~/anaconda3/lib/python3.8/site-packages/minio/api.py in make_bucket(self, bucket_name, location, object_lock)
        624             SubElement(element, "LocationConstraint", location)
        625             body = getbytes(element)
    --> 626         self._url_open(
        627             "PUT",
        628             location,


    ~/anaconda3/lib/python3.8/site-packages/minio/api.py in _url_open(self, method, region, bucket_name, object_name, body, headers, query_params, preload_content, no_body_trace)
        387             self._region_map.pop(bucket_name, None)
        388 
    --> 389         raise response_error
        390 
        391     def _execute(


    S3Error: S3 operation failed; code: BucketAlreadyOwnedByYou, message: Your previous request to create the named bucket succeeded and you already own it., resource: /my-bucket, request_id: 16A553E786AA3496, host_id: 1fd209d2-9d84-4231-8bf0-da6051d4978b, bucket_name: my-bucket


Pour supprimer un bucket, 


```python
client.remove_bucket("my-bucket")
```

Pour lister l'ensemble des buckets présents


```python
client.list_buckets()
```




    [Bucket('my-bucket1'),
     Bucket('my-bucket10'),
     Bucket('my-bucket2'),
     Bucket('my-bucket3'),
     Bucket('my-bucket4'),
     Bucket('my-bucket5'),
     Bucket('my-bucket6'),
     Bucket('my-bucket7'),
     Bucket('my-bucket8'),
     Bucket('my-bucket9')]




```python
for i in range(10): 
    client.make_bucket(f"my-bucket{i+1}")
```


    ---------------------------------------------------------------------------

    S3Error                                   Traceback (most recent call last)

    <ipython-input-24-98d1d3fa9b98> in <module>
          1 for i in range(10):
    ----> 2     client.make_bucket(f"my-bucket{i+1}")
    

    ~/anaconda3/lib/python3.8/site-packages/minio/api.py in make_bucket(self, bucket_name, location, object_lock)
        624             SubElement(element, "LocationConstraint", location)
        625             body = getbytes(element)
    --> 626         self._url_open(
        627             "PUT",
        628             location,


    ~/anaconda3/lib/python3.8/site-packages/minio/api.py in _url_open(self, method, region, bucket_name, object_name, body, headers, query_params, preload_content, no_body_trace)
        387             self._region_map.pop(bucket_name, None)
        388 
    --> 389         raise response_error
        390 
        391     def _execute(


    S3Error: S3 operation failed; code: BucketAlreadyOwnedByYou, message: Your previous request to create the named bucket succeeded and you already own it., resource: /my-bucket1, request_id: 16A54DC754BF6CCF, host_id: 1fd209d2-9d84-4231-8bf0-da6051d4978b, bucket_name: my-bucket1



```python
buckets = client.list_buckets()
for bucket in buckets:
    print(bucket.name, bucket.creation_date)
    #client.remove_bucket(bucket.name)
```

A l'instanciation il est souvent intéressant de vérifier qu'un bucket existe bien.


```python
if client.bucket_exists("my-bucket"):
    print("my-bucket exists")
else:
    print("my-bucket does not exist")
```

    my-bucket does not exist


### Objets

Les objets sont la structure la plus fine de stockage dans Minio. C'est un concept un peu différent des fichiers. Ils regroupent les fichiers ainsi que la structure de dossiers parents. 
Pour donner un exemple l'objet avec le nom `/dossier_parent/dossier_enfant/nom_de_lobjet` integrera aussi la création et le stockage dans les deux dossiers `dossier_parent` et ̀`dossier_enfant`. Pour gérer la structure de rangement des objets il faudra donc aussi jouer sur l'ensemble du chemin vers ce dernier.

On peut stocker n'importe quel fichier ou média dans minio. Une bonne pratique est de préciser le content type qui correspond au type de document stocké. Cela permet au client qui va lire ensuite ce document de savoir comment réagir, est ce que c'est un pdf ? Si oui, je dois l'afficher d'une certaine facon avec potentiellement plusieurs pages, est ce que c'est une video ? Si oui, je dois afficher un lecteur. 

### Ajouter des objets 


```python
import io
import requests
import urllib
```

#### Bytes


```python
result = client.put_object(
    "my-bucket", "my-bytes-object", io.BytesIO(b"hello"), 5,
)
```

#### Csv 


```python
url_velib = "https://www.data.gouv.fr/fr/datasets/r/0845c838-6f18-40c3-936f-da204107759a"

# Upload unknown sized data.
data = urllib.request.urlopen(url_velib)
result = client.put_object(
    "my-bucket", "velib-data", data, length=-1, part_size=10*1024*1024, content_type="application/csv",
)
```

Une image, on peut ajouter des métadatas. 


```python
image_url = "https://media.istockphoto.com/photos/couple-relax-on-the-beach-enjoy-beautiful-sea-on-the-tropical-island-picture-id1160947136?k=20&m=1160947136&s=612x612&w=0&h=TdExAS2--H3tHQv2tc5woAl7e0zioUVB5dbIz6At0I4="
# Upload unknown sized data.
data = urllib.request.urlopen(image_url)
result = client.put_object(
    "my-bucket", "image-plage", data, length=-1, part_size=10*1024*1024, content_type="image/jpeg",
    metadata={"description": "une belle plage"},
)
```


```python

```

### Exercices

1. Importer une vidéo 

### Lister des objets


```python
# List objects information.
objects = client.list_objects("my-bucket")
for obj in objects:
    print(obj, obj.object_name)
```

    <minio.datatypes.Object object at 0x7f6c47da48e0> image-plage
    <minio.datatypes.Object object at 0x7f6c47da4be0> my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4c70> velib-data
    <minio.datatypes.Object object at 0x7f6c47da4c40> path_0/
    <minio.datatypes.Object object at 0x7f6c47da4bb0> path_1/
    <minio.datatypes.Object object at 0x7f6c47da4af0> path_10/
    <minio.datatypes.Object object at 0x7f6c47da4130> path_11/
    <minio.datatypes.Object object at 0x7f6c47da4e50> path_12/
    <minio.datatypes.Object object at 0x7f6c47da49d0> path_13/
    <minio.datatypes.Object object at 0x7f6c47da4fd0> path_14/
    <minio.datatypes.Object object at 0x7f6c47da4a30> path_15/
    <minio.datatypes.Object object at 0x7f6c47da47c0> path_16/
    <minio.datatypes.Object object at 0x7f6c479a6b20> path_17/
    <minio.datatypes.Object object at 0x7f6c479a6eb0> path_18/
    <minio.datatypes.Object object at 0x7f6c479a6190> path_19/
    <minio.datatypes.Object object at 0x7f6c479a6280> path_2/
    <minio.datatypes.Object object at 0x7f6c479a6640> path_20/
    <minio.datatypes.Object object at 0x7f6c479a68b0> path_21/
    <minio.datatypes.Object object at 0x7f6c479a6730> path_22/
    <minio.datatypes.Object object at 0x7f6c47d9e460> path_23/
    <minio.datatypes.Object object at 0x7f6c47d9e580> path_24/
    <minio.datatypes.Object object at 0x7f6c47d9e490> path_25/
    <minio.datatypes.Object object at 0x7f6c47ffd160> path_26/
    <minio.datatypes.Object object at 0x7f6c47ffd8b0> path_27/
    <minio.datatypes.Object object at 0x7f6c47ffd850> path_28/
    <minio.datatypes.Object object at 0x7f6c47ffdaf0> path_29/
    <minio.datatypes.Object object at 0x7f6c47ffd0d0> path_3/
    <minio.datatypes.Object object at 0x7f6c47ffd970> path_4/
    <minio.datatypes.Object object at 0x7f6c47ffd040> path_5/
    <minio.datatypes.Object object at 0x7f6c47ffd070> path_6/
    <minio.datatypes.Object object at 0x7f6c47ffd3d0> path_7/
    <minio.datatypes.Object object at 0x7f6c47ffd8e0> path_8/
    <minio.datatypes.Object object at 0x7f6c47ffd7c0> path_9/



```python
for i in range(30):
    for j in range(30):
        result = client.put_object(
            "my-bucket", f"path_{i}/my-bytes-object-{j}", io.BytesIO(f"hello {i} and {j}".encode()), 13,
        )
```

On peut lister l'ensemble des objets correspondants à un certain prefix


```python
# List objects information.
objects = client.list_objects("my-bucket", recursive=True)
for obj in objects:
    print(obj, obj.object_name)
```

    <minio.datatypes.Object object at 0x7f6c47becd30> image-plage
    <minio.datatypes.Object object at 0x7f6c47bec520> my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47becf70> path_0/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bbb0> path_0/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b970> path_0/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bc40> path_0/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b6d0> path_0/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b6a0> path_0/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b040> path_0/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bc70> path_0/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b070> path_0/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b0a0> path_0/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b820> path_0/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bb50> path_0/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bc10> path_0/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b670> path_0/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417ba00> path_0/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bb80> path_0/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b880> path_0/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bd00> path_0/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bd30> path_0/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bcd0> path_0/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bbe0> path_0/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417ba60> path_0/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b130> path_0/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417baf0> path_0/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417ba90> path_0/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bd90> path_0/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417bf10> path_0/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b9d0> path_0/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b9a0> path_0/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417b910> path_0/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5417be20> path_0/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c5417b940> path_0/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c5417b8e0> path_0/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c5417b610> path_0/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c5417b640> path_0/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c5417b700> path_0/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c5417b7f0> path_0/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c5417bee0> path_0/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c5417b760> path_0/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c5417b850> path_0/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c5417beb0> path_0/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c5417b8b0> path_0/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c5417bf70> path_0/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c5417bdc0> path_0/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c5417bd60> path_0/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c5417bfd0> path_0/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47bd8430> path_0/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47bd83a0> path_0/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47bd8040> path_0/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47bd8370> path_0/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47bd8250> path_0/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47bd82b0> path_0/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47bd8dc0> path_0/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47bd8d60> path_0/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47bd8fa0> path_0/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47bd8cd0> path_0/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47bd8220> path_0/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47bd81c0> path_0/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47bd8f70> path_0/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47bd8280> path_0/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47bd8070> path_1/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8df0> path_1/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8e50> path_1/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8310> path_1/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd83d0> path_1/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd81f0> path_1/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8160> path_1/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8340> path_1/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd82e0> path_1/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8d90> path_1/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8ca0> path_1/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8d00> path_1/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8b50> path_1/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8c10> path_1/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8f40> path_1/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8c70> path_1/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8e20> path_1/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8790> path_1/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8eb0> path_1/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8970> path_1/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8940> path_1/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd89a0> path_1/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8a90> path_1/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8bb0> path_1/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd86d0> path_1/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8820> path_1/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8a60> path_1/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8a30> path_1/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8ac0> path_1/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8af0> path_1/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47bd8b80> path_1/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47bd87c0> path_1/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47bd86a0> path_1/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47bd8610> path_1/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47bd8f10> path_1/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47bd8760> path_1/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47bd8130> path_1/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47bd8460> path_1/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47bd8670> path_1/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47bd87f0> path_1/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47bd8100> path_1/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c476eaf10> path_1/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c476ea040> path_1/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c476ea9a0> path_1/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c476eadc0> path_1/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c476ea5e0> path_1/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c476ea4c0> path_1/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c476ea910> path_1/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c476ea460> path_1/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c476ea8e0> path_1/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c476eaa00> path_1/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c476eafd0> path_1/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c476eabb0> path_1/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c476ea700> path_1/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c476ead30> path_1/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c476eaee0> path_1/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c476eaf40> path_1/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c476eaaf0> path_1/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c476eaac0> path_1/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c476ea970> path_1/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c476ead00> path_10/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea7c0> path_10/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea7f0> path_10/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ead90> path_10/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476eac70> path_10/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea640> path_10/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea400> path_10/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea790> path_10/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea610> path_10/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea3d0> path_10/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea250> path_10/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea430> path_10/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea130> path_10/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea070> path_10/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476eaca0> path_10/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea580> path_10/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea2b0> path_10/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea820> path_10/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea850> path_10/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476eaa60> path_10/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476eaa30> path_10/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea490> path_10/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea6a0> path_10/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476ea4f0> path_10/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f070> path_10/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799fca0> path_10/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799fe20> path_10/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799fb20> path_10/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f400> path_10/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799fd60> path_10/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799fd90> path_10/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c4799fdc0> path_10/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c4799ff70> path_10/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c4799f790> path_10/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c4799f640> path_10/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c4799f7c0> path_10/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c4799f760> path_10/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c4799f970> path_10/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c4799f940> path_10/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c4799fbb0> path_10/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c4799fc70> path_10/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c4799f820> path_10/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c4799f5b0> path_10/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c4799f8b0> path_10/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c4799f880> path_10/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c4799f3d0> path_10/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c4799f460> path_10/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c4799f490> path_10/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c4799f4c0> path_10/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c4799f220> path_10/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c4799f280> path_10/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c4799f2b0> path_10/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c4799f2e0> path_10/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c4799f0d0> path_10/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c4799f0a0> path_10/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c4799f130> path_10/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c4799f040> path_10/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c4799fcd0> path_10/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c4799f6a0> path_10/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c4799fee0> path_10/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c4799fac0> path_11/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f310> path_11/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f340> path_11/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f4f0> path_11/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f520> path_11/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4799f190> path_11/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c69d0> path_11/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c68b0> path_11/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6e80> path_11/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6220> path_11/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6ca0> path_11/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6c70> path_11/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6970> path_11/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6eb0> path_11/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6d30> path_11/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6f10> path_11/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6e50> path_11/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6cd0> path_11/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6af0> path_11/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6c40> path_11/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6d00> path_11/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6760> path_11/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6790> path_11/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6880> path_11/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c68e0> path_11/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c6940> path_11/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5461a3a0> path_11/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e1d430> path_11/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e1d6a0> path_11/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e1d1f0> path_11/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47d9e580> path_11/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47d9e490> path_11/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47d9e550> path_11/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47d9e460> path_11/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47d9ecd0> path_11/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47d9eca0> path_11/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c540fd460> path_11/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c540fd400> path_11/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c540fd190> path_11/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c540fd4c0> path_11/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c540fd7c0> path_11/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c540fd280> path_11/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c540fd430> path_11/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c479a6220> path_11/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c479a6700> path_11/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c479a6fd0> path_11/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c479a6610> path_11/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c479a6fa0> path_11/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c479a6b20> path_11/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c479a64f0> path_11/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c479a6040> path_11/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c479a62b0> path_11/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c479a68e0> path_11/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c479a6250> path_11/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c479a6bb0> path_11/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c479a6eb0> path_11/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c479a6310> path_11/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c479a6e80> path_11/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c479a6490> path_11/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c479a6e20> path_11/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c479a67c0> path_12/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a6400> path_12/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a6280> path_12/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a6190> path_12/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a6640> path_12/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a66d0> path_12/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a6dc0> path_12/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a64c0> path_12/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4be0> path_12/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4400> path_12/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da47c0> path_12/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4f10> path_12/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4160> path_12/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4970> path_12/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4a30> path_12/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4850> path_12/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4130> path_12/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4e50> path_12/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4f40> path_12/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da48e0> path_12/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4ee0> path_12/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4af0> path_12/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da47f0> path_12/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da49d0> path_12/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4fd0> path_12/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54510700> path_12/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54510970> path_12/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c545109a0> path_12/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47d9bb20> path_12/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47d9b670> path_12/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47d9bfd0> path_12/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47d9b400> path_12/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47d9b520> path_12/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47d9b550> path_12/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47d9bd90> path_12/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47d9bd30> path_12/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47d9ba90> path_12/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47d9b3d0> path_12/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c476c62e0> path_12/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c476c6a00> path_12/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c476c6910> path_12/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c476c60a0> path_12/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c476c6100> path_12/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c476c6130> path_12/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c476c6160> path_12/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c476c6370> path_12/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c476c6340> path_12/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c476c6400> path_12/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c476c63d0> path_12/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c476c6dc0> path_12/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c476c6b50> path_12/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c476c6f70> path_12/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c476c6d90> path_12/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c476c6190> path_12/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c476c61c0> path_12/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c476c67c0> path_12/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c476c67f0> path_12/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c476c8b80> path_12/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c476c8b50> path_12/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c476c84f0> path_12/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c476c86d0> path_13/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8a90> path_13/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8ac0> path_13/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8730> path_13/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8880> path_13/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c87c0> path_13/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8af0> path_13/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c80d0> path_13/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8c70> path_13/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8a60> path_13/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8b20> path_13/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8940> path_13/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8910> path_13/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8c10> path_13/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8340> path_13/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8bb0> path_13/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8be0> path_13/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c88e0> path_13/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8f10> path_13/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c89a0> path_13/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8970> path_13/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8eb0> path_13/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8d60> path_13/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8e80> path_13/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8ee0> path_13/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8760> path_13/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8580> path_13/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8640> path_13/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8790> path_13/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8a30> path_13/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c8c40> path_13/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c476c88b0> path_13/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c476c8850> path_13/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c476c8700> path_13/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c476c8250> path_13/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c476c82e0> path_13/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c476c85e0> path_13/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c476c8160> path_13/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c476c80a0> path_13/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c476c8280> path_13/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c476c8130> path_13/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c476c8f70> path_13/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c476c8fa0> path_13/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c476c8400> path_13/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c476c8490> path_13/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c476c8550> path_13/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c476c8460> path_13/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c476c8df0> path_13/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c476c86a0> path_13/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c476c8190> path_13/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c476c81c0> path_13/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c476c2160> path_13/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c476c24c0> path_13/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c476c2af0> path_13/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c476c2310> path_13/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c476c2d00> path_13/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c476c2670> path_13/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c476c2eb0> path_13/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c476c2f10> path_13/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c476c2f40> path_13/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c476c2f70> path_14/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2d60> path_14/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2d30> path_14/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2dc0> path_14/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2cd0> path_14/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2b80> path_14/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c26d0> path_14/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2be0> path_14/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2b50> path_14/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2520> path_14/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2550> path_14/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2640> path_14/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c26a0> path_14/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c23a0> path_14/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c22e0> path_14/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c24f0> path_14/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2370> path_14/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2130> path_14/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2190> path_14/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c21c0> path_14/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c21f0> path_14/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2df0> path_14/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2e20> path_14/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2fa0> path_14/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2fd0> path_14/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2760> path_14/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2580> path_14/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2c40> path_14/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2730> path_14/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2220> path_14/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476c2250> path_14/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c476c23d0> path_14/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c476c2400> path_14/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c476c20a0> path_14/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c476db190> path_14/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c476db4f0> path_14/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c476dbdc0> path_14/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c476dbf70> path_14/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c476dbc10> path_14/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c476db850> path_14/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c476dba00> path_14/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c476db670> path_14/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c476db6d0> path_14/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c476db700> path_14/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c476db730> path_14/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c476db880> path_14/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c476dbfd0> path_14/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c476db8e0> path_14/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c476db820> path_14/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c476dbe50> path_14/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c476dbd90> path_14/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c476dbfa0> path_14/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c476dbe20> path_14/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c476dbbe0> path_14/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c476dbc40> path_14/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c476dbc70> path_14/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c476dbca0> path_14/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c476dba90> path_14/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c476db550> path_14/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c476dbaf0> path_14/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c476dba60> path_15/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db3a0> path_15/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db3d0> path_15/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db4c0> path_15/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db520> path_15/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db220> path_15/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db160> path_15/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db370> path_15/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db1f0> path_15/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476dbe80> path_15/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476dbeb0> path_15/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db040> path_15/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db070> path_15/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476dbb50> path_15/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db910> path_15/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476dbd00> path_15/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476dbb20> path_15/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db5b0> path_15/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db5e0> path_15/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db760> path_15/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db790> path_15/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db280> path_15/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db0a0> path_15/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db430> path_15/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c476db250> path_15/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7a30> path_15/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7760> path_15/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7e20> path_15/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7d90> path_15/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7c70> path_15/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7d60> path_15/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47da7dc0> path_15/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47da7820> path_15/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47da7700> path_15/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47da7670> path_15/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47da7580> path_15/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47da78e0> path_15/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47da77c0> path_15/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47da7a60> path_15/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47da7790> path_15/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47da74f0> path_15/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47da7340> path_15/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47da7400> path_15/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47da7940> path_15/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47da79d0> path_15/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47da7880> path_15/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47da77f0> path_15/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47da7cd0> path_15/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47da7970> path_15/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47da79a0> path_15/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47da7610> path_15/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47da70d0> path_15/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47da76a0> path_15/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47da7190> path_15/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47da75b0> path_15/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47da7640> path_15/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47da7040> path_15/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47da70a0> path_15/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47da73d0> path_15/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47da7160> path_15/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47da7070> path_16/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da75e0> path_16/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da76d0> path_16/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7490> path_16/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7550> path_16/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7430> path_16/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7370> path_16/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7520> path_16/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da73a0> path_16/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7250> path_16/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7220> path_16/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7e50> path_16/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7130> path_16/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da72e0> path_16/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7280> path_16/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7b50> path_16/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7730> path_16/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7e80> path_16/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da78b0> path_16/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7a90> path_16/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7850> path_16/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7ca0> path_16/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da71f0> path_16/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7310> path_16/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7fa0> path_16/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7eb0> path_16/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7460> path_16/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7a00> path_16/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da71c0> path_16/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da7b20> path_16/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da72b0> path_16/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c474dd670> path_16/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c474ddbb0> path_16/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c474ddf70> path_16/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c5417df70> path_16/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c5417dd90> path_16/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c5417dd30> path_16/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c5417df10> path_16/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c5417dee0> path_16/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c5417df40> path_16/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c5417dcd0> path_16/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c5417da00> path_16/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c5417dd60> path_16/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47fe3730> path_16/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47fe38e0> path_16/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47fe3070> path_16/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47fe3580> path_16/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47fe3fa0> path_16/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47fe33d0> path_16/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47fe3c40> path_16/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47fe3df0> path_16/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47fe3e50> path_16/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47fe3e80> path_16/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47fe3f70> path_16/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47fe3fd0> path_16/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47fe3cd0> path_16/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47fe3c10> path_16/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47fe3e20> path_16/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47fe3ca0> path_16/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47fe3a60> path_16/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47fe3ac0> path_17/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3af0> path_17/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3b20> path_17/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3910> path_17/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3790> path_17/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3970> path_17/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe38b0> path_17/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe35e0> path_17/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3610> path_17/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3700> path_17/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3760> path_17/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe35b0> path_17/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3430> path_17/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3280> path_17/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe32b0> path_17/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe33a0> path_17/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3400> path_17/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3100> path_17/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3040> path_17/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3250> path_17/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe30d0> path_17/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3d00> path_17/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3d30> path_17/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3eb0> path_17/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3ee0> path_17/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe39d0> path_17/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe37f0> path_17/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3b80> path_17/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe39a0> path_17/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe3490> path_17/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fe34c0> path_17/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47fe3640> path_17/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47fe3670> path_17/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47fe3160> path_17/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47fe3310> path_17/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47fe3130> path_17/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47db96a0> path_17/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47db9df0> path_17/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47db96d0> path_17/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47db9be0> path_17/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47db9b20> path_17/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47db9970> path_17/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47db9a00> path_17/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47db9c40> path_17/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47db9700> path_17/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47db9b50> path_17/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47db9a30> path_17/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47db99a0> path_17/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47db9370> path_17/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47db9f10> path_17/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47db9430> path_17/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47db9dc0> path_17/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47db9d90> path_17/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47db9e80> path_17/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47db94c0> path_17/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47db9220> path_17/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47db9d60> path_17/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47db9ca0> path_17/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47db9af0> path_17/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47db9490> path_17/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47db95b0> path_18/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db90a0> path_18/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9790> path_18/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9670> path_18/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db95e0> path_18/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9460> path_18/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9610> path_18/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9580> path_18/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9640> path_18/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db94f0> path_18/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9910> path_18/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db97f0> path_18/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9880> path_18/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db98e0> path_18/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9730> path_18/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9250> path_18/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9820> path_18/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9850> path_18/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9100> path_18/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9130> path_18/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db90d0> path_18/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9040> path_18/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db93d0> path_18/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db92b0> path_18/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9340> path_18/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db93a0> path_18/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db91f0> path_18/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db92e0> path_18/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9310> path_18/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9d00> path_18/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47db9d30> path_18/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47db9c70> path_18/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47db9a60> path_18/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47db9fa0> path_18/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47db9ee0> path_18/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47db9bb0> path_18/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47db9fd0> path_18/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47db9eb0> path_18/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47db9cd0> path_18/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47fedbb0> path_18/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47fed9d0> path_18/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47fedd90> path_18/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47fed7c0> path_18/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47fed250> path_18/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47fed280> path_18/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47fed3d0> path_18/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47fed220> path_18/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47fed0d0> path_18/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47fed040> path_18/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47fed700> path_18/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47fed8b0> path_18/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47fed3a0> path_18/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47fed550> path_18/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47fed5b0> path_18/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47fed5e0> path_18/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47fedc70> path_18/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47fede80> path_18/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47fed790> path_18/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47fed6d0> path_18/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47fed580> path_18/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47fed760> path_19/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedcd0> path_19/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedd00> path_19/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedd30> path_19/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedd60> path_19/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedeb0> path_19/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedb20> path_19/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedf10> path_19/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fede50> path_19/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed970> path_19/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed9a0> path_19/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47feda90> path_19/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fedaf0> path_19/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed430> path_19/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed370> path_19/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed940> path_19/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed400> path_19/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed7f0> path_19/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed610> path_19/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47feda00> path_19/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed490> path_19/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed2b0> path_19/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed640> path_19/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed460> path_19/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed130> path_19/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed2e0> path_19/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fed100> path_19/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd86a0> path_19/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8340> path_19/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8310> path_19/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd82e0> path_19/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47fd8190> path_19/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47fd8730> path_19/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47fd8d00> path_19/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47fd89d0> path_19/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47fd8af0> path_19/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47fd8d30> path_19/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47fd88b0> path_19/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47fd8b80> path_19/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47fd8e20> path_19/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47fd8df0> path_19/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47fd8eb0> path_19/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47fd8880> path_19/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47fd8850> path_19/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47fd8820> path_19/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47fd8c10> path_19/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47fd8c40> path_19/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47fd8ee0> path_19/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47fd8be0> path_19/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47fd8b20> path_19/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47fd8fd0> path_19/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47fd8bb0> path_19/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47fd8ac0> path_19/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47fd8a00> path_19/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47fd8b50> path_19/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47fd8a90> path_19/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47fd8a30> path_19/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47fd8910> path_19/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47fd8a60> path_19/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47fd8970> path_19/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47fd8670> path_2/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd83a0> path_2/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8940> path_2/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8520> path_2/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8370> path_2/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8070> path_2/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8490> path_2/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd83d0> path_2/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8f10> path_2/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd89a0> path_2/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8040> path_2/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8d60> path_2/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd86d0> path_2/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8550> path_2/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd88e0> path_2/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8580> path_2/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd84c0> path_2/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8460> path_2/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8790> path_2/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8430> path_2/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8640> path_2/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8610> path_2/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8dc0> path_2/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8d90> path_2/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8cd0> path_2/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47fd8e50> path_2/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997430> path_2/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479974c0> path_2/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997280> path_2/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997250> path_2/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997b50> path_2/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47997640> path_2/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47997610> path_2/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47997760> path_2/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47997460> path_2/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47997730> path_2/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47997550> path_2/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47997490> path_2/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47997e20> path_2/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47997520> path_2/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c479979a0> path_2/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47997910> path_2/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c479974f0> path_2/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47997be0> path_2/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c479978e0> path_2/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c479978b0> path_2/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47997940> path_2/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47997850> path_2/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c479977f0> path_2/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47997370> path_2/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47997880> path_2/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47997820> path_2/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c479976a0> path_2/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47997a00> path_2/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47997340> path_2/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47997700> path_2/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47997790> path_2/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c479972e0> path_2/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47997c10> path_2/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c479977c0> path_2/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47997ac0> path_20/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997580> path_20/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479972b0> path_20/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47997af0> path_20/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479975b0> path_20/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981a60> path_20/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981250> path_20/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981ca0> path_20/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981580> path_20/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479818e0> path_20/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981670> path_20/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479816d0> path_20/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479814c0> path_20/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981460> path_20/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479811c0> path_20/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981490> path_20/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479819a0> path_20/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981940> path_20/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981c40> path_20/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981bb0> path_20/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981790> path_20/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981730> path_20/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981760> path_20/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981850> path_20/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981970> path_20/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981af0> path_20/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981b80> path_20/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981b50> path_20/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479817f0> path_20/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981880> path_20/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47981910> path_20/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47981b20> path_20/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c479812b0> path_20/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47981a00> path_20/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47981820> path_20/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47981070> path_20/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47981130> path_20/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c479814f0> path_20/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c479819d0> path_20/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47981160> path_20/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47981520> path_20/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c479a4e80> path_20/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c479a4ca0> path_20/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c479a4e20> path_20/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c479a4400> path_20/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c479a4760> path_20/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c479a4370> path_20/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c479a43d0> path_20/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c479a4250> path_20/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c479a4070> path_20/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c479a4730> path_20/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c479a4040> path_20/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c479a4fd0> path_20/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c479a4fa0> path_20/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c479a40d0> path_20/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c479a4550> path_20/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c479a4be0> path_20/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c479a4c10> path_20/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c479a4d60> path_20/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c479a4d00> path_20/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c479a4a90> path_21/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4af0> path_21/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4ac0> path_21/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4b80> path_21/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4670> path_21/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4640> path_21/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a48b0> path_21/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4880> path_21/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a41f0> path_21/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a41c0> path_21/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a44f0> path_21/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a44c0> path_21/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4f40> path_21/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4f10> path_21/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4f70> path_21/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4a30> path_21/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4cd0> path_21/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4ee0> path_21/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4340> path_21/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4a00> path_21/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4b50> path_21/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4820> path_21/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4bb0> path_21/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4c40> path_21/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a45b0> path_21/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4460> path_21/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a47f0> path_21/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a45e0> path_21/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4130> path_21/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4dc0> path_21/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c479a4430> path_21/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c479a4160> path_21/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c479a42b0> path_21/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c479a4d90> path_21/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c479a4280> path_21/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47e26820> path_21/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47e26cd0> path_21/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47e266a0> path_21/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47e26430> path_21/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47e26850> path_21/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47e26130> path_21/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47e26c70> path_21/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47e26d00> path_21/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47e26d30> path_21/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47e26a60> path_21/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47e26a90> path_21/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47e26c10> path_21/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47e26520> path_21/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47e26e20> path_21/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47e26fd0> path_21/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47e26790> path_21/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47e26ca0> path_21/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47e26760> path_21/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47e26490> path_21/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47e26fa0> path_21/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47e26880> path_21/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47e268b0> path_21/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47e26070> path_21/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47e26220> path_21/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47e26550> path_21/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47e26bb0> path_22/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26b20> path_22/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26700> path_22/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26df0> path_22/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e262e0> path_22/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26460> path_22/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e269a0> path_22/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26f40> path_22/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26040> path_22/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26d60> path_22/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26970> path_22/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26be0> path_22/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26340> path_22/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47e26d90> path_22/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a670> path_22/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aa00> path_22/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a970> path_22/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aca0> path_22/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ad60> path_22/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aa90> path_22/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ae80> path_22/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aaf0> path_22/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ac10> path_22/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a6d0> path_22/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ae20> path_22/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aee0> path_22/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aac0> path_22/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ad30> path_22/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ae50> path_22/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416adc0> path_22/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416af10> path_22/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c5416aeb0> path_22/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c5416a610> path_22/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c5416a580> path_22/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c5416a4f0> path_22/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c5416a5e0> path_22/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c5416a460> path_22/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c5416a520> path_22/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c5416a550> path_22/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c5416a4c0> path_22/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c5416a820> path_22/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c5416a850> path_22/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c5416a1f0> path_22/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c5416a760> path_22/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c5416aa60> path_22/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c5416a7f0> path_22/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c5416a700> path_22/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c5416abe0> path_22/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c5416a7c0> path_22/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c5416a160> path_22/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c5416a9a0> path_22/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c5416a8e0> path_22/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c5416a640> path_22/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c5416acd0> path_22/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c5416a1c0> path_22/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c5416a790> path_22/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c5416a9d0> path_22/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c5416a8b0> path_22/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c5416abb0> path_22/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c5416a880> path_22/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c5416ad00> path_23/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416aa30> path_23/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a220> path_23/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ad90> path_23/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a5b0> path_23/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416a6a0> path_23/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ab50> path_23/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5416ab20> path_23/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0f40> path_23/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0040> path_23/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be08e0> path_23/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0b80> path_23/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be07c0> path_23/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be08b0> path_23/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0220> path_23/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be05e0> path_23/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0130> path_23/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0310> path_23/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be09a0> path_23/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0d60> path_23/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0a90> path_23/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0c70> path_23/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0400> path_23/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be06d0> path_23/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0b50> path_23/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0a60> path_23/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0e50> path_23/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be04f0> path_23/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0670> path_23/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be06a0> path_23/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47be0c10> path_23/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47be0a00> path_23/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47be0fa0> path_23/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47be0e80> path_23/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47be0f10> path_23/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47be0bb0> path_23/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47be0f70> path_23/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47be0cd0> path_23/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47d86580> path_23/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c5467e610> path_23/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c5467e430> path_23/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c5467e400> path_23/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c5467e7c0> path_23/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c5467edf0> path_23/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c54506790> path_23/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c54506760> path_23/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c54506880> path_23/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c545067f0> path_23/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c54506e20> path_23/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c54506d00> path_23/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c54506b20> path_23/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c5448b550> path_23/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c5448bfa0> path_23/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c5448bf40> path_23/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c5448b8e0> path_23/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c5448be20> path_23/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c5448b580> path_23/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c5448be50> path_23/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c5448b2e0> path_23/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c5448bdf0> path_23/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c5448b160> path_24/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bbe0> path_24/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b6d0> path_24/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bf10> path_24/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bee0> path_24/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bf70> path_24/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b820> path_24/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b9d0> path_24/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bb50> path_24/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b9a0> path_24/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b700> path_24/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b460> path_24/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448bb20> path_24/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b850> path_24/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b250> path_24/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b7c0> path_24/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b3a0> path_24/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b370> path_24/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b610> path_24/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b3d0> path_24/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5448b0a0> path_24/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750dfd0> path_24/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750db20> path_24/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750df10> path_24/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750db80> path_24/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750da30> path_24/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750de50> path_24/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750d8e0> path_24/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750ddc0> path_24/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750d940> path_24/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4750dac0> path_24/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c4750dbe0> path_24/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c4750dc70> path_24/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c4750dd00> path_24/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c4750dcd0> path_24/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c4750d970> path_24/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c4750d9d0> path_24/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c4750df70> path_24/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c4750dc10> path_24/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c4750dee0> path_24/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c4750d9a0> path_24/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c4750df40> path_24/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c4750de80> path_24/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c4750dca0> path_24/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c4750db50> path_24/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c4750ddf0> path_24/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c54688820> path_24/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c54688250> path_24/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c546880d0> path_24/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c54688520> path_24/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c54688850> path_24/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c54688fd0> path_24/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c54688e20> path_24/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c54688550> path_24/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c546889a0> path_24/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c54688e50> path_24/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c546889d0> path_24/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c54688ca0> path_24/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c546880a0> path_24/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c546886a0> path_24/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c54688220> path_25/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c546883d0> path_25/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54688fa0> path_25/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815220> path_25/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c478158e0> path_25/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815df0> path_25/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815e20> path_25/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c478158b0> path_25/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815670> path_25/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815940> path_25/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815040> path_25/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c478156a0> path_25/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815f70> path_25/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815a60> path_25/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47815910> path_25/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c477532e0> path_25/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753250> path_25/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753100> path_25/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753160> path_25/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753e80> path_25/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c477537f0> path_25/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753c40> path_25/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c477530a0> path_25/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c477534c0> path_25/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47753fd0> path_25/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319250> path_25/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319340> path_25/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553192b0> path_25/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553193a0> path_25/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319310> path_25/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553192e0> path_25/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c55319b50> path_25/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c55319460> path_25/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c55319940> path_25/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c553194c0> path_25/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c55319c40> path_25/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c553194f0> path_25/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c55319a60> path_25/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c55319580> path_25/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c55319bb0> path_25/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c553195e0> path_25/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c55319ac0> path_25/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c55319550> path_25/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c55319040> path_25/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c55319670> path_25/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c55319280> path_25/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c55319880> path_25/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c55319220> path_25/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c55319eb0> path_25/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c553191f0> path_25/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c553190a0> path_25/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c55319370> path_25/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c553191c0> path_25/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c553193d0> path_25/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c553190d0> path_25/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c55319400> path_25/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c553197c0> path_25/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c55319490> path_25/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c553196a0> path_25/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c55319430> path_25/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c55319730> path_26/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319520> path_26/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319850> path_26/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553195b0> path_26/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319760> path_26/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319610> path_26/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319fa0> path_26/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319640> path_26/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319910> path_26/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319e50> path_26/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319700> path_26/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319160> path_26/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553196d0> path_26/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319190> path_26/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319790> path_26/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319100> path_26/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319820> path_26/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319070> path_26/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c55319130> path_26/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd760> path_26/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd430> path_26/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd400> path_26/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffdc10> path_26/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd460> path_26/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffdf70> path_26/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd6d0> path_26/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffdb80> path_26/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffdb50> path_26/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd2e0> path_26/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd820> path_26/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffd9a0> path_26/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47ffdfa0> path_26/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47ffdb20> path_26/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47ffd700> path_26/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47ffd070> path_26/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47ffd8e0> path_26/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47ffddf0> path_26/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47ffdbb0> path_26/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47ffdca0> path_26/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47ffdd00> path_26/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47ffd9d0> path_26/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47ffd730> path_26/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47ffd610> path_26/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47ffd4f0> path_26/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47ffd6a0> path_26/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47ffd1c0> path_26/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47ffd520> path_26/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47ffd250> path_26/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47ffd4c0> path_26/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47ffd910> path_26/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47ffd220> path_26/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47ffd3d0> path_26/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47ffd7f0> path_26/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47ffd8b0> path_26/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47ffd7c0> path_26/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47ffd790> path_26/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47ffd940> path_26/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47ffd040> path_26/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47ffd5b0> path_26/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47ffd670> path_26/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47ffd5e0> path_27/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47ffda00> path_27/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7e250> path_27/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7e9d0> path_27/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7eb50> path_27/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7eb80> path_27/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7e070> path_27/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7eb20> path_27/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47f7e190> path_27/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684f10> path_27/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684bb0> path_27/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684ee0> path_27/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684d30> path_27/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684a30> path_27/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684a00> path_27/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684880> path_27/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684430> path_27/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684d00> path_27/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54684580> path_27/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468c160> path_27/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468c610> path_27/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468cd90> path_27/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468cbe0> path_27/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468cf10> path_27/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468c760> path_27/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468c190> path_27/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468cd60> path_27/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c5468c8e0> path_27/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c567823a0> path_27/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c56782400> path_27/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c567826a0> path_27/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c56782520> path_27/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c56782670> path_27/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c567822b0> path_27/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c567826d0> path_27/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c567825b0> path_27/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47daf940> path_27/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47dafac0> path_27/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47daf9d0> path_27/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47daf8e0> path_27/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47dafd30> path_27/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47daf910> path_27/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47dafc40> path_27/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47dafd00> path_27/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47dafca0> path_27/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c47dafe80> path_27/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c47daf370> path_27/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c47daff40> path_27/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c47daf880> path_27/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c47dafeb0> path_27/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c47daf610> path_27/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c47dafee0> path_27/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c47daf490> path_27/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c47dafe50> path_27/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c47daf520> path_27/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c47dafa30> path_27/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c47daf7c0> path_27/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c47dafbe0> path_27/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c47daf040> path_27/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c47dafa90> path_27/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c47daf0d0> path_28/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafa60> path_28/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf220> path_28/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafcd0> path_28/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf0a0> path_28/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafd60> path_28/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf1f0> path_28/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafc70> path_28/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf250> path_28/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafb80> path_28/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf2e0> path_28/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf580> path_28/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf100> path_28/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf730> path_28/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafa00> path_28/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf430> path_28/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafb20> path_28/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf5b0> path_28/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf5e0> path_28/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf820> path_28/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daffd0> path_28/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf8b0> path_28/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf3a0> path_28/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf790> path_28/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf2b0> path_28/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf340> path_28/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafaf0> path_28/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf4c0> path_28/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf3d0> path_28/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47daf850> path_28/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47dafb50> path_28/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c47daf550> path_28/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c47dafdc0> path_28/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c47daf310> path_28/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c47daf6d0> path_28/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c47daf130> path_28/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c47daff10> path_28/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47daf700> path_28/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c540c9e20> path_28/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c553f7ca0> path_28/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c553f7a30> path_28/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c553f79d0> path_28/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c553f7fd0> path_28/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c553f7df0> path_28/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c553f7a60> path_28/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c553f7b20> path_28/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c553f7820> path_28/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c553f7670> path_28/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c553f7fa0> path_28/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c553f7520> path_28/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c553f7700> path_28/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c553f7bb0> path_28/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c553f7eb0> path_28/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c553f75b0> path_28/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c553f7cd0> path_28/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c553f76d0> path_28/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c553f74c0> path_28/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c553f78b0> path_28/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c553f7c40> path_28/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c553f7f40> path_28/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c553f7c70> path_29/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7e20> path_29/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f77f0> path_29/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f75e0> path_29/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7850> path_29/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7a90> path_29/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7790> path_29/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7d60> path_29/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7be0> path_29/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7dc0> path_29/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7d90> path_29/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c553f7c10> path_29/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4b50> path_29/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c47da4910> path_29/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490070> path_29/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c544904c0> path_29/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490640> path_29/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c544901c0> path_29/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490160> path_29/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c544906a0> path_29/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c544905b0> path_29/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490040> path_29/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490400> path_29/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490100> path_29/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490250> path_29/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490220> path_29/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490520> path_29/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c544902e0> path_29/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490670> path_29/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490280> path_29/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c54490370> path_29/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c54490460> path_29/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c54490340> path_29/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c544907f0> path_29/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c54490430> path_29/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c54490610> path_29/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c54490580> path_29/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c47bece20> path_29/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c47becf10> path_29/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c47beca00> path_29/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c47bec1f0> path_29/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c47becd00> path_29/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c47bec4f0> path_29/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c47bec3a0> path_29/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c47bec430> path_29/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c5677d5b0> path_29/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c5677d3d0> path_29/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c5677deb0> path_29/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c5677d220> path_29/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c5677d5e0> path_29/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c5677da60> path_29/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c54622af0> path_29/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c54622b50> path_29/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c474fd610> path_29/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c474fdb20> path_29/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c474fdb80> path_29/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c474fd880> path_29/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c474fd760> path_29/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c474fd460> path_29/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c474fd280> path_29/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c474fd250> path_3/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd670> path_3/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd9a0> path_3/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fda90> path_3/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fda60> path_3/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd640> path_3/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdf10> path_3/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdee0> path_3/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fde50> path_3/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd9d0> path_3/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd8b0> path_3/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdb50> path_3/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd5b0> path_3/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd160> path_3/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd700> path_3/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd130> path_3/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdc10> path_3/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd730> path_3/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd8e0> path_3/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd970> path_3/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd910> path_3/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd940> path_3/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd1f0> path_3/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd2e0> path_3/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd340> path_3/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd2b0> path_3/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdca0> path_3/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fdc40> path_3/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd070> path_3/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd0d0> path_3/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fd040> path_3/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c474fd220> path_3/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c474fd790> path_3/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c474fdac0> path_3/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c474fd7c0> path_3/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c474fd850> path_3/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c474fda30> path_3/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c474fd820> path_3/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c474fd580> path_3/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c474fd6a0> path_3/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c474fd490> path_3/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c474fd400> path_3/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c474fd310> path_3/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c474fd520> path_3/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c474fd430> path_3/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c474fd550> path_3/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c474fd6d0> path_3/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c474fd7f0> path_3/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c474fd4c0> path_3/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c474fd4f0> path_3/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c474fd3d0> path_3/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c474fd3a0> path_3/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c474fd1c0> path_3/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c474fdaf0> path_3/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c474fd0a0> path_3/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c474fd100> path_3/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c474fdbe0> path_3/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c474fd370> path_3/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c474fdcd0> path_3/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c474fde20> path_3/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c474fde80> path_4/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26340> path_4/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e268e0> path_4/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e261c0> path_4/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e260d0> path_4/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26d60> path_4/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26550> path_4/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e264f0> path_4/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26070> path_4/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26250> path_4/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26970> path_4/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26d90> path_4/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26c70> path_4/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e260a0> path_4/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26ca0> path_4/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26460> path_4/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26be0> path_4/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26910> path_4/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26880> path_4/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26640> path_4/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26310> path_4/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e268b0> path_4/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26790> path_4/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26e50> path_4/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26190> path_4/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e266a0> path_4/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26b80> path_4/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26400> path_4/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26220> path_4/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26130> path_4/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e26040> path_4/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c46e26e80> path_4/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c46e26eb0> path_4/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c46e26f40> path_4/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c46e26f10> path_4/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c46e26ee0> path_4/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c46e26f70> path_4/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c46e26fd0> path_4/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c46e26fa0> path_4/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c46e263a0> path_4/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c46e26c10> path_4/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c46e26d00> path_4/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c46e267f0> path_4/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c4666d1c0> path_4/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c4666dcd0> path_4/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c4666d8b0> path_4/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c4666dd30> path_4/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c4666d490> path_4/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c4666dca0> path_4/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c4666d190> path_4/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c4666d220> path_4/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c4666df10> path_4/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c4666d130> path_4/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c4666d4c0> path_4/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c4666d0a0> path_4/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c4666de80> path_4/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c4666d7f0> path_4/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c4666d8e0> path_4/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c4666deb0> path_4/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c4666d670> path_4/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c4666d5e0> path_5/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666d730> path_5/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666dfa0> path_5/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666d5b0> path_5/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666df40> path_5/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666da90> path_5/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c4666dac0> path_5/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9a90> path_5/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9040> path_5/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9f10> path_5/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f91c0> path_5/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f95e0> path_5/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f98e0> path_5/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9ee0> path_5/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9160> path_5/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9ac0> path_5/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9910> path_5/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9580> path_5/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9f40> path_5/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9490> path_5/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9220> path_5/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9070> path_5/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9bb0> path_5/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9730> path_5/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9dc0> path_5/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9340> path_5/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9430> path_5/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9610> path_5/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9520> path_5/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9640> path_5/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474f9310> path_5/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c474f9d00> path_5/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c474f9970> path_5/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c474f9d30> path_5/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c474f9a60> path_5/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c474f9e20> path_5/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c474f9b50> path_5/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c474f9c40> path_5/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c474f9880> path_5/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c474f9ca0> path_5/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c474f9eb0> path_5/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c474f9fd0> path_5/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c474f9be0> path_5/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c474fb250> path_5/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c474fb1f0> path_5/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c474fb100> path_5/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c474fbf70> path_5/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c474fbdf0> path_5/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c474fbe20> path_5/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c474fbd90> path_5/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c474fbdc0> path_5/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c474fbc70> path_5/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c474fba00> path_5/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c474fba30> path_5/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c474fb970> path_5/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c474fb9d0> path_5/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c474fb850> path_5/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c474fb880> path_5/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c474fbca0> path_5/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c474fbcd0> path_5/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c474fbd00> path_6/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbd30> path_6/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbd60> path_6/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbe50> path_6/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbe80> path_6/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbf10> path_6/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbee0> path_6/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbeb0> path_6/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbf40> path_6/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbfd0> path_6/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbfa0> path_6/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbb80> path_6/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbbb0> path_6/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbb20> path_6/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbb50> path_6/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb820> path_6/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb8b0> path_6/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbbe0> path_6/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fbc10> path_6/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fba60> path_6/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fba90> path_6/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb8e0> path_6/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb910> path_6/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb0a0> path_6/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb940> path_6/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb670> path_6/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb5b0> path_6/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb4f0> path_6/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb2e0> path_6/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb430> path_6/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c474fb610> path_6/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c474fb460> path_6/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c474fb400> path_6/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c474fb6a0> path_6/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c474fb9a0> path_6/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c474fb7f0> path_6/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c474fb220> path_6/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c474fb340> path_6/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c474fb790> path_6/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c474fb760> path_6/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c474fb7c0> path_6/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c474fb4c0> path_6/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c474fb550> path_6/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c474fb0d0> path_6/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c474fb2b0> path_6/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c474fb3d0> path_6/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c474fbac0> path_6/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c474fb520> path_6/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c474fb490> path_6/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c474fb700> path_6/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c474fb580> path_6/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c474fb6d0> path_6/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c474fbaf0> path_6/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c474fb370> path_6/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c474fb3a0> path_6/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c46e34550> path_6/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c46e34880> path_6/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c46e34280> path_6/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c46e34370> path_6/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c46e346d0> path_6/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c46e34400> path_7/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34490> path_7/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34430> path_7/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e343d0> path_7/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34ee0> path_7/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34460> path_7/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e345b0> path_7/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e341c0> path_7/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e341f0> path_7/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34160> path_7/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34190> path_7/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34040> path_7/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34070> path_7/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34220> path_7/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e34250> path_7/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e340a0> path_7/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e340d0> path_7/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e335e0> path_7/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e334c0> path_7/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33970> path_7/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33400> path_7/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33ac0> path_7/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33820> path_7/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33850> path_7/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33880> path_7/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e338b0> path_7/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e337c0> path_7/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e337f0> path_7/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33760> path_7/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33790> path_7/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e330d0> path_7/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c46e33100> path_7/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c46e33070> path_7/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c46e330a0> path_7/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c46e33fa0> path_7/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c46e33fd0> path_7/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c46e33e80> path_7/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c46e33eb0> path_7/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c46e33e20> path_7/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c46e33e50> path_7/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c46e33d00> path_7/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c46e33d30> path_7/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c46e33ca0> path_7/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c46e33cd0> path_7/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c46e33b80> path_7/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c46e33bb0> path_7/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c46e33b20> path_7/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c46e33b50> path_7/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c46e33a00> path_7/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c46e33a30> path_7/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c46e339a0> path_7/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c46e339d0> path_7/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c46e33700> path_7/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c46e33730> path_7/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c46e336a0> path_7/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c46e336d0> path_7/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c46e33040> path_7/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c46e33130> path_7/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c46e33160> path_7/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c46e331f0> path_7/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c46e331c0> path_8/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33190> path_8/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33220> path_8/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e332b0> path_8/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33280> path_8/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33ee0> path_8/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33f10> path_8/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33d60> path_8/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33d90> path_8/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33be0> path_8/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33c10> path_8/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33a60> path_8/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33a90> path_8/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e338e0> path_8/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e33910> path_8/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27970> path_8/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27a00> path_8/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e277f0> path_8/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e271c0> path_8/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27250> path_8/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27bb0> path_8/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27be0> path_8/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27490> path_8/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27670> path_8/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27820> path_8/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27b20> path_8/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27610> path_8/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27550> path_8/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e272e0> path_8/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e278b0> path_8/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e277c0> path_8/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c46e271f0> path_8/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c46e27b50> path_8/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c46e27d00> path_8/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c46e274c0> path_8/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c46e279d0> path_8/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c46e27760> path_8/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c46e279a0> path_8/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c46e27a60> path_8/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c46e27a30> path_8/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c46e27640> path_8/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c46e278e0> path_8/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c46e27c40> path_8/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c46e27b80> path_8/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c46e270d0> path_8/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c46e273d0> path_8/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c46e27730> path_8/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c46e275e0> path_8/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c46e276d0> path_8/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c46e27310> path_8/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c46e274f0> path_8/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c46e27130> path_8/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c46e27400> path_8/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c46e27220> path_8/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c46e27910> path_8/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c46e27100> path_8/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c46e27850> path_8/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c46e27520> path_8/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c46e27a90> path_8/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c46e27ac0> path_8/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c46e275b0> path_9/0-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27580> path_9/1-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27430> path_9/10-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27460> path_9/11-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27af0> path_9/12-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e276a0> path_9/13-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27790> path_9/14-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27700> path_9/15-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27280> path_9/16-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e273a0> path_9/17-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27160> path_9/18-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27340> path_9/19-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27190> path_9/2-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e272b0> path_9/20-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27d30> path_9/21-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27d60> path_9/22-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27ee0> path_9/23-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27f10> path_9/24-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27e80> path_9/25-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27e50> path_9/26-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27d90> path_9/27-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27dc0> path_9/28-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27df0> path_9/29-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27e20> path_9/3-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27eb0> path_9/4-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27f40> path_9/5-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27f70> path_9/6-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27fa0> path_9/7-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27fd0> path_9/8-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27ca0> path_9/9-my-bytes-object
    <minio.datatypes.Object object at 0x7f6c46e27cd0> path_9/my-bytes-object-0
    <minio.datatypes.Object object at 0x7f6c46e27c10> path_9/my-bytes-object-1
    <minio.datatypes.Object object at 0x7f6c46e315b0> path_9/my-bytes-object-10
    <minio.datatypes.Object object at 0x7f6c46e31970> path_9/my-bytes-object-11
    <minio.datatypes.Object object at 0x7f6c46e316a0> path_9/my-bytes-object-12
    <minio.datatypes.Object object at 0x7f6c46e31e20> path_9/my-bytes-object-13
    <minio.datatypes.Object object at 0x7f6c46e311f0> path_9/my-bytes-object-14
    <minio.datatypes.Object object at 0x7f6c46e313d0> path_9/my-bytes-object-15
    <minio.datatypes.Object object at 0x7f6c46e31c40> path_9/my-bytes-object-16
    <minio.datatypes.Object object at 0x7f6c46e31f10> path_9/my-bytes-object-17
    <minio.datatypes.Object object at 0x7f6c46e312e0> path_9/my-bytes-object-18
    <minio.datatypes.Object object at 0x7f6c46e31a60> path_9/my-bytes-object-19
    <minio.datatypes.Object object at 0x7f6c46e31790> path_9/my-bytes-object-2
    <minio.datatypes.Object object at 0x7f6c46e31d30> path_9/my-bytes-object-20
    <minio.datatypes.Object object at 0x7f6c46e31880> path_9/my-bytes-object-21
    <minio.datatypes.Object object at 0x7f6c46e314c0> path_9/my-bytes-object-22
    <minio.datatypes.Object object at 0x7f6c46e31b50> path_9/my-bytes-object-23
    <minio.datatypes.Object object at 0x7f6c46e31af0> path_9/my-bytes-object-24
    <minio.datatypes.Object object at 0x7f6c46e317f0> path_9/my-bytes-object-25
    <minio.datatypes.Object object at 0x7f6c46e31cd0> path_9/my-bytes-object-26
    <minio.datatypes.Object object at 0x7f6c46e31d00> path_9/my-bytes-object-27
    <minio.datatypes.Object object at 0x7f6c46e31df0> path_9/my-bytes-object-28
    <minio.datatypes.Object object at 0x7f6c46e31820> path_9/my-bytes-object-29
    <minio.datatypes.Object object at 0x7f6c46e31e80> path_9/my-bytes-object-3
    <minio.datatypes.Object object at 0x7f6c46e31910> path_9/my-bytes-object-4
    <minio.datatypes.Object object at 0x7f6c46e31940> path_9/my-bytes-object-5
    <minio.datatypes.Object object at 0x7f6c46e31700> path_9/my-bytes-object-6
    <minio.datatypes.Object object at 0x7f6c46e31760> path_9/my-bytes-object-7
    <minio.datatypes.Object object at 0x7f6c46e319a0> path_9/my-bytes-object-8
    <minio.datatypes.Object object at 0x7f6c46e31a90> path_9/my-bytes-object-9
    <minio.datatypes.Object object at 0x7f6c46e318b0> velib-data



```python
objects = client.list_objects("my-bucket", prefix="path_10/")
for obj in objects:
    print(obj.object_name)
```

    path_10/0-my-bytes-object
    path_10/1-my-bytes-object
    path_10/10-my-bytes-object
    path_10/11-my-bytes-object
    path_10/12-my-bytes-object
    path_10/13-my-bytes-object
    path_10/14-my-bytes-object
    path_10/15-my-bytes-object
    path_10/16-my-bytes-object
    path_10/17-my-bytes-object
    path_10/18-my-bytes-object
    path_10/19-my-bytes-object
    path_10/2-my-bytes-object
    path_10/20-my-bytes-object
    path_10/21-my-bytes-object
    path_10/22-my-bytes-object
    path_10/23-my-bytes-object
    path_10/24-my-bytes-object
    path_10/25-my-bytes-object
    path_10/26-my-bytes-object
    path_10/27-my-bytes-object
    path_10/28-my-bytes-object
    path_10/29-my-bytes-object
    path_10/3-my-bytes-object
    path_10/4-my-bytes-object
    path_10/5-my-bytes-object
    path_10/6-my-bytes-object
    path_10/7-my-bytes-object
    path_10/8-my-bytes-object
    path_10/9-my-bytes-object
    path_10/my-bytes-object-0
    path_10/my-bytes-object-1
    path_10/my-bytes-object-10
    path_10/my-bytes-object-11
    path_10/my-bytes-object-12
    path_10/my-bytes-object-13
    path_10/my-bytes-object-14
    path_10/my-bytes-object-15
    path_10/my-bytes-object-16
    path_10/my-bytes-object-17
    path_10/my-bytes-object-18
    path_10/my-bytes-object-19
    path_10/my-bytes-object-2
    path_10/my-bytes-object-20
    path_10/my-bytes-object-21
    path_10/my-bytes-object-22
    path_10/my-bytes-object-23
    path_10/my-bytes-object-24
    path_10/my-bytes-object-25
    path_10/my-bytes-object-26
    path_10/my-bytes-object-27
    path_10/my-bytes-object-28
    path_10/my-bytes-object-29
    path_10/my-bytes-object-3
    path_10/my-bytes-object-4
    path_10/my-bytes-object-5
    path_10/my-bytes-object-6
    path_10/my-bytes-object-7
    path_10/my-bytes-object-8
    path_10/my-bytes-object-9


### Créer des URLS présigner 

Les méthodes précédentes sont très intéressantes pour communiquer entre une API et Minio. L'inconvéniant est que le média doit forcément passer par le serveur. Pour des raisons de performances réseaux, il est beaucoup plus intéressant d'utiliser les clients pour envoyer directement les données volumineuses.

Pour cela il faut permettre aux clients de s'authentifier directement sur Minio. Evidemment on ne peut pas donner aux clients (Les utilisateurs mobiles ou navigateurs les informations de connexion qui peuvent être stockés dans l'API), on utilisera alors des URLs présignés. Ces URLs un peu complexes à lire intègrent directement plusieurs informations permettant au client de s'authentifier. Pour des raisons de sécurité il est aussi important de rendre ces URLs inutilisable au bout d'un certain temps. Les clés générées sont à usage unique. 

Il faut alors créer des URLs présigner pour uploader des médias mais aussi pour y accéder. Ces URLs doivent être générés par l'API. Les clients doivent demander à l'API, "puis-je poster un média ?" ou "puis-je voir ce média ?" si la réponse est oui , alors l'API génère un URL qui peut ensuite être utilisé pour réaliser l'action.


```python
from datetime import timedelta
```

Pour envoyer un média


```python
put_url  = client.get_presigned_url(
    "PUT",
    "my-bucket",
    "my-object",
    expires=timedelta(days=1),
    response_headers={"response-content-type": "application/json"},
)
print(url)
```

    http://localhost:9000/my-bucket/my-object?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=root%2F20210916%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210916T132448Z&X-Amz-Expires=7200&X-Amz-SignedHeaders=host&X-Amz-Signature=c98daa553660fc8b20424301f190d14c76506d7c3fe288728f66cb53630f0786



```python
import requests
```


```python
response = requests.put(put_url, json={"key":"value"})
response
```




    <Response [200]>



Pour lire un média.


```python
get_url = client.get_presigned_url(
    "GET",
    "my-bucket",
    "my-object",
    expires=timedelta(hours=2),
)
print(url)
```

    http://localhost:9000/my-bucket/my-object?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=root%2F20210916%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20210916T132448Z&X-Amz-Expires=7200&X-Amz-SignedHeaders=host&X-Amz-Signature=c98daa553660fc8b20424301f190d14c76506d7c3fe288728f66cb53630f0786



```python
response = requests.get(get_url)
response.json()
```




    {'key': 'value'}



## Exercices API

1. Ajouter une route permettant de créer un URL présigné PUT
2. Ajouter une route permettant de récupérer un URL présigné GET en fonction d'un document
3. Ajouter une route permettant de lister l'ensemble des objets présents dans un bucket.
4. Ajouter un paramètre permettant de rendre ce listing reccursif. 


```python

```
