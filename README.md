# Tractorbeam Backup

Open source, multi-tier website backups to S3.

Tractorbeam is a Docker container that can backup data from a variety of sources to an S3-compatible object store such as AWS, DigitalOcean Spaces, Google Cloud, Ceph, and more.  All libraries and utilities are already in the container, so there's nothing more to install.

## Multi-tier backups

This container also creates "multi-tier" backups in a best-practice fashion. Instead of creating only a linear series of backups, Tractorbeam creates a directory structure:

```
my/custom/prefix
└── db-backups
    ├── daily
    │   ├── flight_deck_test_db_20XX0101120000.sql.gz
    │   ...
    │   └── flight_deck_test_db_20XX0107120000.sql.gz
    ├── monthly
    └── weekly
```

Each directory only retains a set number of files by default:
* **daily** stores daily backups, up to the last seven days.
* **weekly** stores weekly backups, taken once a week, up to the last 4 weeks.
* **monthly** stores monthly backups, taken once a week, up to the last 12 months.

This reduces the overall amount of storage you require, while still providing you sufficient coverage for auditing or disaster recovery.

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

For Docker Compose, you can attach a `docker-compose run` command to the Docker host's normal cronjob system.

## Configuration

This container does not use environment variables for configuration. Instead, it relies on the `tractorbeam.yml` file:

```yaml
---
tractorbeam:
  databases: {}
  platformshDatabases: {}
```

Where:

* **databases** is a list of locally accessible MySQL/MariaDB databases to back up to S3.
* **platformshDatabases** is a list of [Platform.sh](https://platform.sh) database relationships to back up to S3.

## Backing up MySQL/MariaDB

Create one or more items under the `tractorbeam.databases` list:

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

The each item describes a MySQL/MariaDB database to backup, where:

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
      accessKeyFile: "/secrets/my_bucket_name_accessKey.txt"
      secretKeyFile: "/secrets/my_bucket_name_secretKey.txt"
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
      accessKey: "abcef123456"
      secretKey: "abcef123456"
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
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Depending on your custom endpoint, you may or may not need to specify a `region` as well.

## Backing up Platform.sh DB relationships

This container can also back up a database relationship to S3:

```yaml
---
platformshDatabases:
  - project: "wxyz0987"
    environment: "master"
    relationship: "my_db_relationship"
    cliToken: "abcdefghijklmnop1234567890"
    identityFile: "/config/psh-backup/id_rsa"
    bucket: "my_bucket_name"
    prefix: "my/custom/prefix"
    accessKey: "abcef123456"
    secretKey: "abcef123456"
    endpoint: "https://sfo2.digitaloceanspaces.com"
```

* **project** is the project ID on Platform.sh. Required.
* **environment** is the environment to back up. Optional, defaults to `master`.
* **relationship** is the name of the database relationship to back up. Required.
* **cliToken** is your [Platform.sh CLI token](https://docs.platform.sh/gettingstarted/cli/api-tokens.html). Optional if `cliTokenFile` is defined.
* **identityFile** is the full path to the private key associated with your Platform.sh account. The public key must be in the same directory. Required.
* **bucket** is the S3 bucket name used to store the backup. Required.
* **prefix** is the prefix to use when saving the backup. Optional, defaults to an empty string.
* **accessKey** is the S3 access key. Optional if `accessKeyFile` is defined.
* **secretKey** is the S3 secret key to the bucket. Optional if `secretKeyFile` is defined.

Note that `identifyFile` must **always** be a file path inside the container.

### Using separate files for credentials

This works the same way as it does for the `tractorbeam.database` item:

```yaml
---
platformshDatabases:
  - project: "wxyz0987"
    environment: "master"
    relationship: "my_db_relationship"
    cliTokenFile: "/config/psh-backup/platform-cli-token.txt"
    identityFile: "/config/psh-backup/id_rsa"
    bucket: "my_bucket_name"
    prefix: "my/custom/prefix"
    accessKeyFile: "/secrets/my_bucket_name_accessKey.txt"
    secretKeyFile: "/secrets/my_bucket_name_secretKey.txt"
    endpoint: "https://sfo2.digitaloceanspaces.com"
```

Where:

* **cliTokenFile** is the full path to a file containing the Platform.sh CLI token.
* **accessKeyFile** is the full path to a file containing the S3 bucket access key.
* **secretKeyFile** is the full path to a file containing the S3 bucket secret key.

All paths are to the file *inside* the container.

## Deployment on Kubernetes

Use the [`ten7.flightdeck_cluster`](https://galaxy.ansible.com/ten7/flightdeck_cluster) role on Ansible Galaxy to deploy Tractorbeam as a series of cronjobs:

```yaml
flightdeck_cluster:
  namespace: "backup"
  configMaps:
    - name: "tractorbeam"
      files:
        - name: "tractorbeam.yml"
          content: |
            tractorbeam:
              databases:
                - name: "flight_deck_test_db"
                  user: "flying_red_panda"
                  passwordFile: "/config/flight-deck-test-backup/db-pass.txt"
                  host: "mysql.database.svc.cluster.local"
                  bucket: "my_bucket_name"
                  prefix: "my/custom/prefix"
                  accessKeyFile: "/config/flight-deck-test-backup/s3-key.txt"
                  secretKeyFile: "/config/flight-deck-test-backup/s3-secret.txt"
    - name: "flight-deck-test-backup"
      files:
        - name: "db-pass.txt"
          content: "weeee"
        - name: "s3-key.txt"
          content: "abcef123456"
        - name: "s3-secret.txt"
          content: "abcef123456"
  tractorbeam:
    configMaps:
      - name: "tractorbeam"
        path: "/config/tractorbeam"
    secrets:
      - name: "flight-deck-test-backup"
        path: "/config/flight-deck-test-backup"
```

## Using with Docker Compose

Create the `tractorbeam.yml` file relative to your `docker-compose.yml`. Define the `tractorbeam` service mounting the file as a volume:

```yaml
version: '3'
services:
  solr:
    image: ten7/tractorbeam
    volumes:
      - ./tractorbeam.yml:/config/tractorbeam/tractorbeam.yml
    command: ["tractorbeam", "daily"]
```

Be sure to change the second item of the `command` to either `daily`, `weekly`, or `monthly` as appropriate to run either a daily, weekly, or monthly backup.

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
