# Lab 02 Cuztomized Appplication Deployment

## Introduction

In the second lab we are going to slightly modify our application. In order to do that we need to 
* do few modifications in the source code of the applications 
* build new docker images, that contain these changes
* push these images to a Docker registry
* deploy applications to Kubernetes

We are going to push images to a private docker registry (Harbor) and authorize Kubernetes cluster to pull images from this registry.

## Private Docker registry

Each workshop participant (ideally) has his own Harbor Docker registry running in his Kubernetes cluster (backed by AWS S3 storage). It must be available at ```https://harbor.svc.<cluster-name>.superkube.kubernetes.delivery```. Let's try to log into the registry and configure our Kubernetes to pull images from the registry.

1. Run ```docker login harbor.svc.<cluster-name>.superkube.kubernetes.delivery```. Enter username and password (admin:Harbor12345). The output should be: ```Login Succeeded```

2. Create a secret that contains credentials of your docker registry:
```kubectl create secret docker-registry harbor-<cluster-name> --docker-server=harbor.svc.<cluster-name>.superkube.kubernetes.delivery --docker-username=admin --docker-password=Harbor12345 --docker-email=viktorsoginskis@gmail.com``` The output should be ```secret "harbor-viktor" created```.

> Objects of type [secret](https://kubernetes.io/docs/concepts/configuration/secret/) are intended to hold sensitive information, such as passwords, OAuth tokens, and ssh keys. Putting this information in a secret is safer and more flexible than putting it verbatim in a pod definition or in a docker image.

3. Check that secret has been created using ```kubectl get secret harbor-<cluster-name> --output=json```. The output should be similar to:
```
{
    "apiVersion": "v1",
    "data": {
        ".dockerconfigjson": "eyJhdXRocyI6eyJoYXJib3Iuc3ZjLnZpa3Rvci5zdXBlcmt1YmUua3ViZXJuZXRlcy5kZWxpdmVyeSI6eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJIYXJib3IxMjM0NSIsImVtYWlsIjoidmlrdG9yc29naW5za2lzQGdtYWlsLmNvbSIsImF1dGgiOiJZV1J0YVc0NlNHRnlZbTl5TVRJek5EVT0ifX19"
    },
    "kind": "Secret",
    "metadata": {
        "creationTimestamp": "2018-09-25T18:15:08Z",
        "name": "harbor-viktor",
        "namespace": "default",
        "resourceVersion": "273070",
        "selfLink": "/api/v1/namespaces/default/secrets/harbor-viktor",
        "uid": "ef98a741-c0ee-11e8-b7db-06d0009de686"
    },
    "type": "kubernetes.io/dockerconfigjson"
}
```
4. cd to k8s-wordsmith-demo directory and modify ```kube-deployment.yml``` file. Add ```imagePullSecrets``` to the ```specs``` section of each ```Deployment``` in order to use the secret we created in Step 2. Example:
```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: db
  labels:
    app: words-db
spec:
  template:
    metadata:
      labels:
        app: words-db
    spec:
      containers:
      - name: db
        image: 
        ports:
        - containerPort: 5432
          name: db
      imagePullSecrets:
      - name: harbor-viktor
```
Now Kubernetes is ready to pull images from private Docker registry!

## Modify the DB application

Let's make our application to build sentences from Latvian words instead of English words!
In order to do so we need to modify a SQL script that inserts data (words) to application's database.

1. cd to k8s-wordsmith-demo directory. It contains sources of 3 Docker images (2 application sources and the database) we deployed in the first lab. 
2. cd to k8s-wordsmith-demo/db
3. The directory contains [words.sql](k8s-wordsmith-demo/db/words.sql) that is executed when the database container gets bootstrapped:
```
CREATE TABLE nouns (word TEXT NOT NULL);
CREATE TABLE verbs (word TEXT NOT NULL);
CREATE TABLE adjectives (word TEXT NOT NULL);

INSERT INTO nouns(word) VALUES
  ('dators'),
  ('koks'),
  ('zīmulis'),
  ('ūdens'),
  ('cilvēks'),
  ('pele');

INSERT INTO verbs(word) VALUES
  ('lieto'),
  ('skaitļo'),
  ('aug'),
  ('raksta'),
  ('ir'),
  ('ēd');

INSERT INTO adjectives(word) VALUES
  ('akls'),
  ('dzidrs'),
  ('centīgs'),
  ('ciets'),
  ('vecs');
```
This file replaces initial words.sql script, that inserted English words.
4. Let's build a new Docker image for the database and push it to the private Docker registry.
