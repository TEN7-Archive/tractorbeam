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

This container does not use environment variables for configuration. Instead, it relies on a `tractorbeam.yml` mounted in `/config`:

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
      accessKey: "abcef123456"
      secretKey: "abcef123456"
```

The each item in the `databases` list describes a MySQL/MariaDB database to backup, where:

* **name** is the name of the database. Required.
* **user** is the user with which to access the database. Required.
* **password** is the password to access the database. Optional if `passwordFile` is defined.
* **host** is the database hostname. Optional, defaults to `localhost`.
* **bucket** is the S3 bucket name used to store the backup. Required.
* **prefix** is the prefix to use when saving the backup. Optional, defaults to an empty string.
* **accessKey** is the S3 access key. Optional if `accessKeyFile` is defined.
* **secretKey** is the S3 secret key to the bucket. Optional if `secretKeyFile` is defined.

### Using separate files for credentials

Sometimes, you may wish to keep certain credentials in separate files from the rest of the configuration. One such case is if you want to keep `tractorbeam.yml` in a Kubernetes configmap, but keep the database password, and bucket keys in secrets instead.

For these files, you can use the `*File` version:

```yaml
---
tractorbeam:
  databases:
    - name: "flight_deck_test_db"
      user: "flying_red_panda"
      passwordFile: "/secrets/panda_password.txt"
      host: "mysql.database.svc.cluster.local"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKeyFile: "/secrets/my_bucket_name_access_key.txt"
      secretKeyFile: "/secrets/my_bucket_name_secret_key.txt"
```

Where:

* **passwordFile** is the full path to a file containing the database password.
* **accessKeyFile** is the full path to a file containing the S3 bucket access key.
* **secretKeyFile** is the full path to a file containing the S3 bucket secret key.

All paths are to the file *inside* the container.

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

In Docker Compose, mount the `tractorbeam.yml` file at `/config/tractorbeam.yml` in the service's `volumes` key.

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
                  mountPath: /config
          restartPolicy: OnFailure
          volumes:
            - name: vol-secrets
              secret:
                secretName: tractorbeam
```

## Part of Flight Deck

This container is part of the [Flight Deck library](https://github.com/ten7/flight-deck) of containers for Drupal local development and production workloads on Docker, Swarm, and Kubernetes.

Flight Deck is used and supported by [TEN7](https://ten7.com/).

## Debugging

If you need to get verbose output from the entrypoint, set `flightdeck_debug` to `true` or `yes` in the config file.

```yaml
---
flightdeck_debug: yes
```

This container uses [Ansible](https://www.ansible.com/) to perform start-up tasks. To get even more verbose output from the start up scripts, set the `ANSIBLE_VERBOSITY` environment variable to `4`.

If the container will not start due to a failure of the entrypoint, set the `FLIGHTDECK_SKIP_ENTRYPOINT` environment variable to `true` or `1`, then restart the container.

## License

Tractorbeam is licensed under GPLv3. See [LICENSE](https://raw.githubusercontent.com/ten7/flight-deck/master/LICENSE) for the complete language.
