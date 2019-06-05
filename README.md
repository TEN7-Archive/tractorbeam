# Tractorbeam

Open source, multi-tier backups to S3.

## Use

This container runs as a single-shot, and does not stay resident intentionally. Instead, you need to run the container regularly as part of a external scheduler. For Kubernetes, define a `cronjob` to run the container:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: tractorbeam_daily
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: tractorbeam
              image: ten7/tractorbeam
              args:
                - tractorbeam
                - daily
          restartPolicy: OnFailure
```

## Configuration

This container does not use environment variables for configuration. Instead, it relies on a `tractorbeam.yml` mounted in `/secrets`:

```yaml
---
tractorbeam:
  databases:
    - name: "flight_deck_test_db"
      user: "flying_red_panda"
      password: "weeee"
      host: "mysql.database.svc.cluster.local"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      access_key: "abcef123456"
      secret_key: "abcef123456"
```

The each item in the `databases` list describes a MySQL/MariaDB database to backup, where:

* **name** is the name of the database.
* **user** is the user with which to access the database.
* **host** is the database hostname. Optional, defaults to `localhost`.
* **bucket** is the S3 bucket name used to store the backup.
* **prefix** is the prefix to use when saving the backup.
* **access_key** is the S3 access key.
* **secret_key** is the S3 secret key to the bucket.

### Specifying the region

If using AWS S3, you should specify the region name in your configuration:

```yaml
---
tractorbeam:
  databases:
    - name: "flight_deck_test_db"
      user: "flying_red_panda"
      password: "weeee"
      host: "mysql.database.svc.cluster.local"
      bucket: "my_bucket_name"
      region: "us-east-1"
      prefix: "my/custom/prefix"
      access_key: "abcef123456"
      secret_key: "abcef123456"
```

### Using alternate endpoints

You may also specify an alternate S3 endpoint to use any S3 compatible object store such as DigitalOcean Spaces, or a Ceph cluster. To do so, include the `endpoint` key:

```yaml
---
tractorbeam:
  databases:
    - name: "flight_deck_test_db"
      user: "flying_red_panda"
      password: "weeee"
      host: "mysql.database.svc.cluster.local"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      access_key: "abcef123456"
      secret_key: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Depending on your custom endpoint, you may or may not need to specify a `region` as well.

### Mounting the configuration

In Docker Compose, mount the `tractorbeam.yml` file at `/secrets/tractorbeam.yml` in the service's `volumes` key.

In Kubernetes, create a `stringData` secret to house the config:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tractorbeam
type: Opaque
stringData:
  tractorbeam.yml: |
    ---
    tractorbeam:
      databases:
        - name: "flight_deck_test_db"
          user: "flying_red_panda"
          password: "weeee"
          host: "mysql.database.svc.cluster.local"
          bucket: "my_bucket_name"
          prefix: "my/custom/prefix"
          access_key: "abcef123456"
          secret_key: "abcef123456"
          endpoint: "https://sfo2.digitaloceanspaces.com"
```

Then, update the cronjob definition to mount the secret:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: tractorbeam_daily
spec:
  schedule: "0 0 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: tractorbeam
              image: ten7/tractorbeam
              args:
                - tractorbeam
                - daily
              volumeMounts:
                - name: vol-secrets
                  mountPath: /secrets
          restartPolicy: OnFailure
          volumes:
            - name: vol-secrets
              secret:
                secretName: tractorbeam
```
