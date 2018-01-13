# Configuring Your Artifact Repository

To run Argo workflows that use artifacts, you must configure and use an artifact repository.
Argo supports any S3 compatible artifact repository such as AWS, GCS and Minio.

# Configuring Minio

```
$ brew install kubernetes-helm # mac
$ helm init
$ helm install stable/minio --name argo-artifacts
```

Login to the Minio UI using a web browser (port 9000) after obtaining the external IP using `kubectl`.
```
$ kubectl get service argo-artifacts-minio-svc
```
On Minikube:
utputs:
      artifacts:
      - name: message
        path: /tmp/hello_world.txt
        s3:
          # Use the corresponding endpoint depending on your S3 provider:
          #   AWS: s3.amazonaws.com
          #   GCS: storage.googleapis.com
          #   Minio: my-minio-endpoint.default:9000
          endpoint: s3.amazonaws.com
          bucket: my-bucket
          # NOTE that all output artifacts are automatically tarred and 
          # gzipped before saving. So as a best practice, .tgz or .tar.gz
          # should be incorporated into the key name so the resulting file
          # has an accurate file extension.
          key: path/in/bucket/hello_world.tgz
          # accessKeySecret and secretKeySecret are secret selectors.
          # It references the k8s secret named 'my-s3-credentials'.
          # This secret is expected to have have the keys 'accessKey'
          # and 'secretKey', containing the base64 encoded credentials
          # to the bucket.
          accessKeySecret:
            name: my-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-s3-credentials
            key: secretKey

```
$ minikube service --url argo-artifacts-minio-svc
```

NOTE: When minio is installed via Helm, it uses the following hard-wired default credentials,
which you will use to login to the UI:
* AccessKey: AKIAIOSFODNN7EXAMPLE
* SecretKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

Create a bucket named `my-bucket` from the Minio UI.


# Configuring AWS S3

Create your bucket and access keys for the bucket. AWS access keys have the same permissions as the user they are associated with. In particular, you cannot create access keys with reduced scope. If you want to limit the permissions for an access key, you will need to create a user with just the permissions you want to associate with the access key. Otherwise, you can just create an access key using your esiting user account.

```
$ export mybucket=bucket249
$ cat > policy.json <<EOF
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:PutObject",
            "s3:GetObject"
         ],
         "Resource":"arn:aws:s3:::$mybucket/*"
      }
   ]
}
EOF
$ aws s3 mb s3://$mybucket [--region xxx]
$ aws iam create-user --user-name $mybucket-user
$ aws iam put-user-policy --user-name $mybucket-user --policy-name $mybucket-policy --policy-document file://policy.json
$ aws iam create-access-key --user-name $mybucket-user > access-key.json
```

# Configuring GCS (Google Cloud Storage)
Create a bucket from the GCP Console (https://console.cloud.google.com/storage/browser).

Enable S3 compatible access and create an access key.
Note that S3 compatible access is on a per project rather than per bucket basis.
- Navigate to Storage > Settings (https://console.cloud.google.com/storage/settings).
- Enable interoperability access if needed.
- Create a new key if needed.

# Using Configured Artifact Repositories
Once the S3 compatible artifact repositories are configured, you can access them from the input and output sections of your workflow template specs.

Use the `endpoint` corresponding to your S3 provider:
- AWS: s3.amazonaws.com
- GCS: storage.googleapis.com
- Minio: my-minio-endpoint.default:9000

The `key` is name of the object in the `bucket` The `accessKeySecret` and `secretKeySecret` are secret selectors that reference the specified kubernetes secret.  The secret is expected to have have the keys 'accessKey' and 'secretKey', containing the base64 encoded credentials to the bucket.

For AWS, the `accessKeySecret` and `secretKeySecret` correspond to AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY respectively.

For GCS, the `accessKeySecret` and `secretKeySecret` for S3 compatible access can be obtained from the GCP Console. Note that S3 compatible access is on a per project rather than per bucket basis.
- Navigate to Storage > Settings (https://console.cloud.google.com/storage/settings).
- Enable interoperability access if needed.
- Create a new key if needed.

For Minio, the `accessKeySecret` and `secretKeySecret` naturally correspond the AccessKey and SecretKey already discussed above.

```
  templates:
  - name: artifact-example
    inputs: 
      artifacts:
      - name: my-input-artifact
        path: /my-input-artifact
        s3:
          endpoint: s3.amazonaws.com
          bucket: my-aws-bucket-name
          key: path/in/bucket/my-input-artifact.tgz
          accessKeySecret:
            name: my-aws-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-aws-s3-credentials
            key: secretKey
    outputs:
      artifacts:
      - name: my-output-artifact
        path: /my-ouput-artifact
        s3:
          endpoint: storage.googleapis.com
          bucket: my-aws-bucket-name
          # NOTE that all output artifacts are automatically tarred and 
          # gzipped before saving. So as a best practice, .tgz or .tar.gz
          # should be incorporated into the key name so the resulting file
          # has an accurate file extension.
          key: path/in/bucket/my-output-artifact.tgz
          accessKeySecret:
            name: my-gcs-s3-credentials
            key: accessKey
          secretKeySecret:
            name: my-gcs-s3-credentials
            key: secretKey
    container:
      image: debian:latest
      command: [sh, -c]
      args: ["cp -r /my-input-artifact /my-output-artifact"]
```