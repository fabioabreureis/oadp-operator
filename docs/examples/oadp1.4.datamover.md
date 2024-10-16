<hr style="height:1px;border:none;color:#333;">
<h1 align="center">OADP 1.4 Kopia + Datamover configuration</h1>
<h2 align="center">Using a generic s3 Bucket</h2>

### Prerequisites
* OADP operator, a credentials secret, and a DataProtectionApplication (DPA) CR 
  are all created. Follow [these steps](/docs/install_olm.md) for installation instructions.

  - Make sure your DPA CR is similar to below in the install step.
  - versions: Openshift 4.14 and OADP 1.4
 

Data protection Application 

```
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: oadp-backup
  namespace: openshift-adp
spec:
  backupLocations:
  - velero:
      config:
        profile: default
        region: netapp
        s3ForcePathStyle: "true"
        s3Url: https://URL-S3-BUCKET
      credential:
        key: cloud
        name: cloud-credentials
      default: true
      objectStorage:
        bucket: BUCKET-NAME
        caCert: xxxx
        prefix: velero
      provider: aws
  configuration:
    nodeAgent:
      enable: true
      uploaderType: kopia
    velero:
      defaultPlugins:
      - openshift
      - aws
      - kubevirt
      - csi
      defaultSnapshotMoveData: true
      featureFlags: 
      - EnableCSI
```

<hr style="height:1px;border:none;color:#333;">

### Create the Nginx deployment:

`oc create -f docs/examples/manifests/nginx/nginx-deployment.yaml`

This will create the following resources:
* **Namespace**
* **Deployment**
* **Service**
* **Route**

### Verify Nginx deployment resources:

`oc get all -n nginx-example`

Should look similar to this:

```
$ oc get all -n nginx-example
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-55ddb59f4c-bls2x   1/1     Running   0          2m9s
pod/nginx-deployment-55ddb59f4c-cqjw8   1/1     Running   0          2m9s

NAME               TYPE           CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)          AGE
service/my-nginx   LoadBalancer   172.30.46.198   aef02efae2e95444eaeef61c92dbc441-1447257770.us-east-2.elb.amazonaws.com   8080:30193/TCP   2m10s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deployment   2/2     2            2           2m10s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deployment-55ddb59f4c   2         2         2       2m10s

NAME                                HOST/PORT                                                              PATH   SERVICES   PORT   WILDCARD
route.route.openshift.io/my-nginx   my-nginx-nginx-example.apps.cluster-da0d.da0d.sandbox591.opentlc.com          my-nginx   8080   None
```

### Create a backup schedule 
schedule.yaml 
```
apiVersion: velero.io/v1
kind: Schedule
metadata: 
  name: schedule-test-oadp
  namespace: openshift-adp
spec:
  schedule: '0 7 * * * '
  skipImmediately: false
  template:
	hooks: {}
	includedNamespaces:
  	- test-oadp
	includedResources:
  	- namespace
  	- deployments
  	- configmaps
  	- secrets
  	- pvc
  	- pv
  	- services
  	- routes
	ttl: 240h0m0s
```

`oc create -f schedule.yaml`

### Create backup 

Create the velero alias 
`alias velero='oc -n openshift-adp exec deployment/velero -c velero -it -- ./velero'`

Create the backup 
`velero create backup --from-schedule schedule-test-oadp datamover`

Verify the backup
`oc get backup -n openshift-adp nginx-stateless -o jsonpath='{.status.phase}'`

should result in `Completed`

During the backup process check if the volume snapshot content and dataupload is created:

`oc get vsc -n openshift-adp`

`oc get datauploads -n openshift-adp`


### Create Restore 

Create the restore 
`velero create restore --from-backup datamover datamover`

Verify the restore
`oc get backup -n openshift-adp nginx-stateless -o jsonpath='{.status.phase}'`

During the backup process check if datadownload is created:
`oc get datadownloads -n openshift-adp`
