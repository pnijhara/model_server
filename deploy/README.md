# Kubernetes deployment

A helm chart for installing OpenVino Model Server on Kubernetes cluster is provided. By default the cluster contains 
a single instance of the inference server but the replicaCount configuration parameter can be set to create a cluster 
of any size, as described below. This guide assumes you already have a functional Kubernetes cluster and helm 
installed (see below for instructions on installing helm).

The steps below describe how to set-up a model repository, use helm to launch the inference server, and then send 
inference requests to the running server. 

## Installing Helm

Please refer to: https://helm.sh/docs/intro/install for Helm installation.

## Model repository

If you already have a model repository you may use that with this helm chart. If you don't, you can use any model
from https://download.01.org/opencv/2020/openvinotoolkit/2020.2/open_model_zoo/models_bin .

OpenVINO Model Server needs models that it will make available for inferencing. For example you can 
use Google Cloud Storage bucket:
```shell script
gsutil mb gs://models-repository
```

You can download the model from the link provided above nad upload it to GCS:
```shell script
gsutil cp -r docs/examples/model_repository gs://triton-inference-server-repository/model_repository
```

## Bucket permissions

Make sure the bucket permissions are set so that the inference server can access the model repository. If the bucket 
is public then no additional changes are needed and you can proceed to "Running The Inference Server" section.

If bucket permissions need to be set with the GOOGLE_APPLICATION_CREDENTIALS environment variable then perform the 
following steps:

* Generate Google service account JSON with proper permissions called gcp-creds.json.
* Create a Kubernetes secret from this file:

      $ kubectl create configmap gcpcreds --from-literal "project-id=myproject"
      $ kubectl create secret generic gcpcreds --from-file gcp-creds.json

* Modify templates/deployment.yaml to include the GOOGLE_APPLICATION_CREDENTIALS environment variable:
      
      env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /secret/gcp-creds.json

* Modify templates/deployment.yaml to mount the secret in a volume at /secret:

      volumeMounts:
        - name: vsecret
          mountPath: "/secret"
          readOnly: true
      ...
      volumes:
      - name: vsecret
        secret:
          secretName: gcpcreds
          
## Deploy the Model Server

Deploy the Model Server using helm:
```shell script
helm install --name ovms ovms
```

Use kubectl to see status and wait until the model server pod is running:
```shell script
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
example-triton-inference-server-5f74b55885-n6lt7   1/1     Running   0          2m21s
```

## Using OpenVINO Model Server

Now that the inference server is running you can send HTTP or GRPC requests to it to perform inferencing. 
By default, the inferencing service is exposed with a LoadBalancer service type. Use the following to find the 
external IP for the inference server. In this case it is 34.83.9.133:
```shell script
$ kubectl get services
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                        AGE
...
example-triton-inference-server  LoadBalancer   10.18.13.28    34.83.9.133   8000:30249/TCP,8001:30068/TCP,8002:32723/TCP   47m
```

The inference server exposes an HTTP endpoint on port 8000, and GRPC endpoint on port 8001 and a Prometheus metrics endpoint on port 8002. You can use curl to get the status of the inference server from the HTTP endpoint:

$ curl 34.83.9.133:8000/api/status
Follow the instructions to get the example image classification client that can be used to perform inferencing using image classification models being served by the inference server. For example:

$ image_client -u 34.83.9.133:8000 -m resnet50_netdef -s INCEPTION -c3 mug.jpg
Request 0, batch size 1
Image 'images/mug.jpg':
    504 (COFFEE MUG) = 0.723992
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115997





## Cleanup

Once you've finished using the inference server you should use helm to delete the deployment:
```shell script
$ helm list
NAME            REVISION  UPDATED                   STATUS    CHART                          APP VERSION   NAMESPACE
example         1         Wed Feb 27 22:16:55 2019  DEPLOYED  triton-inference-server-1.0.0  1.0           default
example-metrics       1               Tue Jan 21 12:24:07 2020        DEPLOYED        prometheus-operator-6.18.0       0.32.0          default

$ helm delete --purge example
$ helm delete --purge example-metrics
```

You may also want to delete the GCS bucket you created to hold the model repository:
```shell script
$ gsutil rm -r gs://models-repository
```