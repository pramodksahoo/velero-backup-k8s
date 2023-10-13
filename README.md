## Velero Server: 
The core component of Velero is the Velero server. It's responsible for managing backup and restore operations in our Kubernetes cluster. It runs as a set of containers in our cluster and interacts with other components and Kubernetes resources.

#### Kubernetes API Server: 
Velero communicates with the Kubernetes API server to perform backup and restore operations. It creates custom resources and backup-related objects to manage backups and restores.

#### Plugin System: 
Velero supports various storage plugins that allow it to interact with different cloud providers(AWS, Azure, DigitalOcean) and storage solutions. These plugins enable Velero to capture and restore data, including persistent volumes.

#### Backup: 
When you trigger a backup operation, Velero takes a snapshot of the specified resources and data within your cluster. This includes both the application state (e.g., Deployments, StatefulSets) and any associated data stored in persistent volumes.

#### Backup Location: 
Backup locations are used to define the target storage location for your Velero backups, Like Local Directory, NFS (Network File System), Cloud Storage(AWS S3 bucket, Azure Blob Storage, DigitalOcean Spaces). When configuring a backup location in Velero, you will typically provide details such as the `storage provider, access credentials, and the bucket or directory` where backups should be stored.
##### Here we are using Cloud Storage `AWS S3 bucket`

#### VolumeSnapshot: 
A volume snapshot location in Velero is a custom resource that defines where persistent volume snapshots (PVSs) should be stored for a given backup. This allows you to store your PVSs in a different location than your backup data, which can be useful for various reasons, such as:

- To store your PVSs in a more durable or cost-effective location.
- To store your PVSs in a different region to improve performance or disaster recovery.
- To store your PVSs with a different provider, such as a cloud provider or on-premises storage.

However, there are a few limitations to keep in mind:
- Velero only supports a single set of credentials per provider. This means that you cannot use different credentials for different volume snapshot locations within the same provider.
- Volume snapshots are still limited by where your provider allows you to create snapshots. For example, AWS and Azure do not allow you to create a volume snapshot in a different region than where the volume is.
- It is not possible to send a single Velero backup to multiple volume snapshot locations simultaneously, or a single volume snapshot to multiple locations simultaneously. However, you can always set up multiple scheduled backups that differ only in the volume snapshot locations used if redundancy of backups across locations is important.
#### Schedule and Retention: 
Velero provides a way to schedule backups at regular intervals and configure retention policies. This ensures that you have a history of backups and can restore to a specific point in time.

#### Restore: 
In the event of a disaster or data loss, Velero can be used to restore your applications and data from a previous backup. You can choose to restore the entire cluster or specific resources.

#### Validation: 
Velero performs validation during restore to ensure that the backup can be successfully restored to the cluster. This includes checking for resource dependencies and compatibility with the current cluster state
## Installing the Velero server
In our Setup we installed velero server in Digitalocean K8s cluster with CI pipeline Helm chart and taking backup with different resources of that cluster(like Namespace, selector, entire cluster) and storing the backup location in AWS S3 bucket.
### Installation Requirements
- Kubernetes v1.16+, because this helm chart uses CustomResourceDefinition `apiextensions.k8s.io/v1`. This API version was introduced in Kubernetes v1.16.

- An AWS IAM user with an access key and secret key and permission to put and get objects in the AWS S3 bucket.

- Create a bucket on AWS S3 to store the backup.
#### Variable need to add for create secret from values
| Variable | Description | Environment | Example |
|---|---|---|---|
| VELERO_S3_BUCKET_NAME | S3 bucket use to store the backup | both | staging-raicoon-backups |
| VELERO_S3_ACCESS_KEY_ID | AWS Access key-ID | both | AW$ACCE$$KEY!D |
| VELERO_S3_SECRET_ACCESS_KEY | AWS Secret key | both | AW$$ECRETKEY |
| DIGITALOCEAN_API_TOKEN | Digitalocean token use for authenticate to k8s cluster | both | D!G1TAL0CETOKEN |

### Velero version

This helm chart installs Velero version v1.12 https://velero.io/docs/v1.12/. See the [#Upgrading](#upgrading) section for information on how to upgrade from other versions.

Need to Update few fields for input of configuration section in values.yaml file
```
configuration:
  backupStorageLocation:
  - name: aws
    provider: velero.io/aws
    bucket:     # Replace with bucket name 
    prefix:     # Replace with bucket path
    default: true
    accessMode: ReadWrite 
    credential:
      name: velero-secrets # secret name 
      key: cloud
    config:
      region:    # Region where your bucket is located
      s3Url: https://s3.amazonaws.com
```
```
credentials:
  useSecret: true
  name: velero-secrets   # Secret Name 
  secretContents:
    cloud: |
      [default]
      aws_access_key_id=${VELERO_S3_ACCESS_KEY_ID}          # define in gitlab CI/CD variables
      aws_secret_access_key=${VELERO_S3_SECRET_ACCESS_KEY}  # define in gitlab CI/CD variables
  # additional key/value pairs to be used as environment variables such as "DIGITALOCEAN_TOKEN: <your-key>". Values will be stored in the secret.
  extraEnvVars: 
    DIGITALOCEAN_TOKEN: ${DIGITALOCEAN_API_TOKEN}
```

Please see the many options supported in the values.yaml file. [Velero Values file](https://github.com/vmware-tanzu/helm-charts/blob/velero-5.1.0/charts/velero/values.yaml).

Installing velero server using Helm chart values.yaml 

`helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts`

`helm install velero vmware-tanzu/velero --namespace velero --create-namespace -f values.yaml`

## Installing the Velero CLI

`wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz`

`tar -xvf velero-v1.12.0-linux-amd64.tar.gz`

Move the extracted velero binary to somewhere in your $PATH (/usr/local/bin for most users).

See the different options for installing the [Velero CLI](https://velero.io/docs/v1.12/basic-install/#install-the-cli).

#### Backup a namespace
`velero backup create <Backup_Name>  --include-namespaces <Namespace_name>`

#### To show backup operation details
`velero describe backup <Backup_Name> --details`
#### List all stored backups (name, status, ..) and get the backup name
`velero get backups`
#### Initiate the restore operation
`velero restore create <Restore_Name> --from-backup <Backup_Name>`

### Backups on schedule
#### Daily Backup, 7 days retention policy with selector
`velero schedule create <schedule_Name-daily-ret-7d> --schedule="0 0 * * *" --ttl 168h0m0s --selector app=duokey`

#### Weekly Backup, 30 days retention policy with selector
`velero schedule create <Schedule_Name-weekly-ret-30d> --schedule="1 0 */7 * *" --ttl 720h0m0s --selector app=duokey`

For Schedule backup we can define in the `values.yaml` to set cron expression to periodically take backup.
``````
# Backup schedules to create.
schedules:
  daily-backup:
    disabled: false
    labels:
      backup-type: daily
    annotations:
      description: "Daily backup of all resources"
    schedule: "0 0 * * *"  # This is a Cron format for daily backups
    useOwnerReferencesInBackup: false
    template:
      ttl: "720h"  # Keep backups for 30 days
      storageLocation: "aws"
      includedNamespaces:
      - '*'  # All namespaces
``````
``````
Note: You could also use the @daily, @monthly, @Yearly Cron notation

velero create schedule <Schedule_Name> --schedule="@every 12h" --include-namespaces <Namespace_Name>
# Check all list schedules
velero schedules get
``````
### Velero PVCs volume backup
Here are some additional things to keep in mind when using volume snapshots in Velero:
 - Volume snapshots are only supported for PVCs that are provisioned using a CSI volume driver.
 - Velero will only back up the data that is stored on the PVC. It will not back up any metadata associated with the PVC.
 - When you restore a Velero backup that used volume snapshots, the new PVC will be created in the same namespace as the original PVC.
 - You cannot restore a Velero backup that used volume snapshots to a different cluster.

1. To take a PVC backup with volume snapshots in Velero, you need to follow these steps:
Note: You must have the `EnableCSI` feature flag enabled in `values.yaml` and the Velero CSI plugins installed in order to use volume snapshots in Velero.
```
initContainers:
  # - name: velero-plugin-for-csi
  #   image: velero/velero-plugin-for-csi:v0.6.0
  #   imagePullPolicy: IfNotPresent
  #   volumeMounts:
  #     - mountPath: /target
  #       name: plugins
```
2. Create a volume snapshot location. This is a custom resource that defines where Velero will store volume snapshots.

Here is an example of how to take a PVC backup with volume snapshots in Velero:

Each object in the volumeSnapshotLocation list must have the following fields:
 - name: The name of the volume snapshot location.
 - provider: The name of the volume snapshot provider.
 - credential: The name of the secret that contains the credentials for the volume snapshot provider.
 - config: A map of provider-specific configuration parameters.

Here is an example of a volumeSnapshotLocation section in a values.yaml file:
```
volumeSnapshotLocation:
- name: my-snapshot-location
  provider: aws
  credential: my-aws-credentials
  config:
    region: us-east-1
```
3. Create a Velero backup using the velero backup command with the --volume-snapshot-location flag.
When you create the Velero backup, Velero will take a snapshot of each PVC in the <my-namespace> namespace. The snapshot will be stored in the volume snapshot location that you specified.
##### Create a Velero backup of pvc volume
`velero backup my-backup --volume-snapshot-location my-snapshot-location --namespace my-namespace`

4. Restore the Velero backup using the velero restore command.
Once the backup is complete, you can restore it using the velero restore command. When you restore the backup, Velero will create a new PVC for each PVC that was backed up. The new PVC will be created using the volume snapshot that was taken during the backup.

##### Restore the Velero volume backup
`velero restore my-backup`

