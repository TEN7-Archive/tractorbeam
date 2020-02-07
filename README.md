# Tractorbeam Backup

Open source, multi-tier website backups to S3.

Tractorbeam is a Docker container that can backup data from a variety of sources to an S3-compatible object store such as AWS, DigitalOcean Spaces, Google Cloud, Ceph, and more.  All libraries and utilities are already in the container, so there's nothing more to install.

## Multi-tier backups

This container also creates "multi-tier" backups in a best-practice fashion. Instead of creating only a linear series of backups, Tractorbeam creates a directory structure:

```
my/custom/prefix
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
tractorbeam:
  databases: {}
  platformshDatabases: {}
  platformshFiles: {}
  archives: {}
  files: {}
  s3: {}
```

Where:

* **databases** is a list of locally accessible MySQL/MariaDB databases to back up to S3.
* **platformshDatabases** is a list of [Platform.sh](https://platform.sh) database relationships to back up to S3.
* **platformshFiles** is a list of [Platform.sh](https://platform.sh) file mounts to back up to S3.
* **archives** is a list of SSH-accessible (SSH, SFTP, rsync-over-ssh) directories from which to create a snapshot archive, and upload to S3.
* **files** is a list SSH-accessible files to perform a rolling directory backup to S3.
* **s3** is a list of backups to duplicate files between S3 buckets.

### Specifying the backup target

All backup types back up to an S3-compatible object store such as AWS S3, DigitalOcean Spaces, Ceph, Google Cloud, etc.. As such, every item in each list has the following items:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
```

* **bucket** is the S3 bucket name used to store the backup. Required.
* **prefix** is the prefix to use when saving the backup. Optional, defaults to an empty string.
* **accessKey** is the S3 access key. Optional if `accessKeyFile` is defined.
* **secretKey** is the S3 secret key. Optional if `secretKeyFile` is defined.


### Using separate files for credentials

Sometimes, you may wish to keep certain credentials in separate files from the rest of the configuration. One such case is if you want to keep `tractorbeam.yml` in a Kubernetes ConfigMap, but store bucket keys, database passwords, SSH keys, and other critical credentials in a Secret instead for additional security.

For these, you can use the `*File` version:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKeyFile: "/secrets/my_bucket_name_accessKey.txt"
      secretKeyFile: "/secrets/my_bucket_name_secretKey.txt"
```

Where:

* **accessKeyFile** is the full path inside the container to a file containing the S3 bucket access key.
* **secretKeyFile** is the full path inside the container to a file containing the S3 bucket secret key.

### Specifying the region

If using AWS S3 for the target bucket, you should specify the region name in your configuration:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      region: "us-east-1"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
```

Where:

* **region** is the AWS S3 region in which your bucket resides. Optional.

### Using alternate endpoints

You may also specify an alternate S3 endpoint to use any S3-compatible object store such as DigitalOcean Spaces, Ceph, Google Cloud, and so on. To do so, include the `endpoint` key:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Where:

* **endpoint** is the S3 API endpoint URL for your S3-compatible object store.

Depending on your custom endpoint, you may or may not need to specify a `region` as well.

### Completion pings

For all backups, Tractorbeam can send a ping to [Healthchecks.io](https://healthchecks.io/) on completion. Healthchecks.io is an open source, periodic task monitoring system. This can be used to track the completion of each backup simply by providing a URL:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
      healthcheckUrl: "https://hc-ping.com/abcdefghijklmnop1234567890"
```

### Disabling a backup

If you need to disable a particular backup, you can use the `disabled` key:

```yaml
tractorbeam:
  exampleBackupType:
    - bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
      healthcheckUrl: "https://hc-ping.com/abcdefghijklmnop1234567890"
      disabled: yes
```

Where:

* **disabled** is if the backup is disabled or not. Uses a YAML boolean.

## Backing up databases

Tractorbeam supports backing the following databases:

* MySQL/MariaDB databases which are network accessible
* Database relationships on Platform.sh

### Backing up MySQL/MariaDB

Create one or more items under the `tractorbeam.databases` list:

```yaml
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

If you prefer to keep the **password** in a separate file, you can use `passwordFile` instead:

```yaml
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

* **passwordFile** is the full path inside the container to a file containing the database password.

### Backing up Platform.sh DB relationships

This container can also back up a database relationship on your Platform.sh project to S3:

```yaml
tractorbeam:
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
* **cliTokenFile** is the full path inside the container to a file containing the [Platform.sh CLI token](https://docs.platform.sh/gettingstarted/cli/api-tokens.html). Optional.
* **cliToken** is your Platform.sh CLI token. Optional if `cliTokenFile` is defined.
* **identityFile** is the full path inside the container to an SSH private key associated with your Platform.sh account. The public key must be in the same directory. Required.

Note that you can associate multiple SSH keys with your Platform.sh account. It is highly recommended to create a dedicated key for Tractorbeam, rather than share your existing key.

## Backing up files

Tractorbeam can download and backup a snapshot of a remote directory. Once downloaded, these files are compressed into a timestamped archive and then uploaded to S3.

Downloading and creating snapshot files is a space and bandwidth intensive process. Avoid using this method for directories where the contents are larger than 300MB. Instead, see "Rolling Directory Backups" below.

Tractorbeam supports the following file archive backups:
* SSH-accessible (SFTP/rsync-over-ssh) sources

### Backing up files over SSH

To backup snapshots of a directory over SSH, create the `tractorbeam.archives` item:

```yaml
tractorbeam:
  archives:
    - src: "example.com"
      user: "myexampleuser"
      path: "/path/to/my/files"
      identityFile: "/config/my-backup/id_rsa"
      archivePrefix: "wbphotonics_code_live_"
      archiveFormat: "gz"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Each item in the list is a SSH/SFTP/rsync-to-S3 snapshot backup to perform, where:

* **src** is the source domain name or IP address. Do **not** include a protocol. Required.
* **user** is the username with which to access the files. Required.
* **path** is the path on the remote server from which to backup files. Required.
* **identityFile** is the full path inside the container to the SSH private key with which to connect to the source server. The public key must be in the same directory. Required.
* **archivePrefix** is the prefix of the snapshot filename to apply before the timestamp and file extension. Required;
* **archiveFormat** is the format to use to create the snapshot archive. Optional, defaults to `gz` for a `*.tar.gz` archive.

The `archiveFormat` can be one of the following values:
* `bz2`
* `gz` (the default)
* `tar` (no compression)
* `xz`
* `zip`

Note, it is best to always create a dedicated SSH key for Tractorbeam, rather than share your existing SSH keys.


## Rolling directory backups

For large directories (>300MB), archiving a new snapshot each time is space and bandwidth intensive. In that case, you may wish to do a "rolling" backup. That is, only the most recent contents of the source directory are preserved, with no archiving or timestamping performed. This is useful for website managed file directories where most changes are new files being added, rather than existing files being modified.

Tractorbeam supports the following rolling directory backups:
* SSH-accessible (SFTP/rsync-over-ssh) sources
* S3 to S3

### Caching files for rolling backups

Backing up files can take a considerable amount of bandwidth to synchronize, especially if the files you're backing up only have additions or rare changes. In those cases, you may use the `cacheDir` key to store any downloaded files within the container:

```yaml
tractorbeam:
  exampleBackupType:
    - cacheDir: "/backups/files/my_bucket_name/my/custom/prefix"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Where:

* **cacheDir** is the full path to the directory inside the container to cache the backups. The directory must exist, and should be mounted as a persistent volume.

### Rolling backup of files over SSH

Tractorbeam can perform a rolling backup an entire directory from any SSH-accessible source. To do so, create one or more items under the `tractorbeam.files` list:

```yaml
tractorbeam:
  files:
    - src: "example.com"
      user: "myexampleuser"
      path: "/path/to/my/files"
      delete: true
      identityFile: "/config/my-backup/id_rsa"
      bucket: "my_bucket_name"
      prefix: "my/custom/prefix"
      accessKey: "abcef123456"
      secretKey: "abcef123456"
      endpoint: "https://sfo2.digitaloceanspaces.com"
```

Each item in the list is a SSH/SFTP/rsync-to-S3 rolling backup to perform, where:

* **src** is the source domain name or IP address. Do **not** include a protocol. Required.
* **user** is the username with which to access the files. Required.
* **path** is the path on the remote server from which to backup files. Required.
* **delete** specifies if files not present in the source directory should be deleted in S3. Optional, defaults to true.
* **identityFile** is the full path inside the container to the SSH private key with which to connect to the source server. The public key must be in the same directory. Required.

Note, it is best to always create a dedicated SSH key for Tractorbeam, rather than share your existing SSH keys.

### Backing up Platform.sh file mounts

This container can also back up a file mount on your Platform.sh project to S3:

```yaml
tractorbeam:
  platformshFiles:
    - project: "wxyz0987"
      environment: "master"
      mount: "web/sites/default/files"
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
* **mount** is the name of the file mount from `platform mounts` to back up. Required.
* **cliTokenFile** is the full path inside the container to a file containing the [Platform.sh CLI token](https://docs.platform.sh/gettingstarted/cli/api-tokens.html). Optional.
* **cliToken** is your Platform.sh CLI token. Optional if `cliTokenFile` is defined.
* **identityFile** is the full path inside the container to an SSH private key associated with your Platform.sh account. The public key must be in the same directory. Required.

Note that you can associate multiple SSH keys with your Platform.sh account. It is highly recommended to create a dedicated key for Tractorbeam, rather than share your existing key.

### Backing up S3 Buckets

Sometimes you may need to mirror the contents of an S3 bucket to another S3 bucket. Tractorbeam can do this too under the `tractorbeam.s3` list:

```yaml
tractorbeam:
  s3:
    - srcBucket: "my_bucket_name"
      srcPrefix: "my/custom/prefix"
      srcAccessKey: "abcef123456"
      srcSecretKey: "abcef123456"
      srcEndpoint: "https://sfo2.digitaloceanspaces.com"
      bucket: "my_redundant_bucket"
      delete: yes
      prefix: "my/custom/prefix"
      accessKey: "vwxyz098765"
      secretKey: "vwxyz098765"
```

Each item in the list is a, s3-to-s3 backup to perform, where:

* **srcBucket** is the source S3 bucket name to use when retrieving files to backup. Required.
* **srcPrefix** is the prefix inside the source bucket for files to backup. Optional, defaults to the root of the bucket.
* **srcAccessKey** is the access key for the source bucket. Optional if `srcAccessKeyFile` is defined.
* **srcSecretKey** is the secret key for the source bucket. Optional if `srcSecretKeyFile` is defined.
* **srcRegion** is the S3 region in which `srcBucket` resides. Optional.
* **srcEndpoint** is the S3 API endpoint to use for the source bucket. Optional, defaults to AWS S3.
* **delete** specifies if files not present in the source bucket should be deleted in the target bucket. Optional, defaults to true.

By design, the S3-to-S3 backup is always performed *last* in Tractorbeam. This allows you to mirror previous backups easily.

## Deployment

Tractorbeam may be deployed in several ways, including base Docker, Docker Compose, Swarm, and Kubernetes.

### Kubernetes

Use the [`ten7.flightdeck_cluster`](https://galaxy.ansible.com/ten7/flightdeck_cluster) role on Ansible Galaxy to deploy Tractorbeam as a series of CronJobs:

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
    size: "100Gi"
    configMaps:
      - name: "tractorbeam"
        path: "/config/tractorbeam"
    secrets:
      - name: "flight-deck-test-backup"
        path: "/config/flight-deck-test-backup"
```

### Docker Compose

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

To mount additional credential files, be sure to add them to the `volumes` key:

```yaml
version: '3'
services:
  solr:
    image: ten7/tractorbeam
    volumes:
      - ./tractorbeam.yml:/config/tractorbeam/tractorbeam.yml
      - ./db_password.txt:/secrets/db_password.txt
    command: ["tractorbeam", "daily"]
```

## Part of Flight Deck

This container is part of the [Flight Deck library](https://github.com/ten7/flight-deck) of containers for Drupal local development and production workloads on Docker, Swarm, and Kubernetes.

Flight Deck is used and supported by [TEN7](https://ten7.com/).

## Debugging

If you need to get verbose output from the entrypoint, set `flightdeck_debug` to `true` or `yes` in the config file.

```yaml
flightdeck_debug: yes
```

This container uses [Ansible](https://www.ansible.com/) to perform start-up tasks. To get even more verbose output from the start up scripts, set the `ANSIBLE_VERBOSITY` environment variable to `4`.

If the container will not start due to a failure of the entrypoint, set the `FLIGHTDECK_SKIP_ENTRYPOINT` environment variable to `true` or `1`, then restart the container.

## License

Tractorbeam is licensed under GPLv3. See [LICENSE](https://raw.githubusercontent.com/ten7/flight-deck/master/LICENSE) for the complete language.
