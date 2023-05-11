# Cloud Pak for Business Automation Production deployment manual installation ✍️<!-- omit in toc -->

For version 22.0.2 IF004

Installs ODM environment.

- [Disclaimer ✋](#disclaimer-)
- [Prerequisites](#prerequisites)
- [Needed tooling](#needed-tooling)
- [Install preparation](#install-preparation)
- [Command line preparation in install Pod](#command-line-preparation-in-install-pod)
- [Prerequisite software](#prerequisite-software)
  - [OpenLDAP](#openldap)
    - [Access info after deployment](#access-info-after-deployment)
  - [PostgreSQL](#postgresql)
    - [Access info after deployment](#access-info-after-deployment-1)
- [Cloud Pak for Business Automation CASE package](#cloud-pak-for-business-automation-case-package)
- [Cloud Pak for Business Automation (ODM) Development Environment](#cloud-pak-for-business-automation-development-environment)
  - [Get certified kubernetes folder](#get-certified-kubernetes-folder)
  - [Cluster setup](#cluster-setup)
  - [Customize CPFS before deployment](#customize-cpfs-before-deployment)
  - [Prepare property files](#prepare-property-files)
  - [Generate, update and apply SQL and Secret files and validate the connectivity](#generate-update-and-apply-sql-and-secret-files-and-validate-the-connectivity)
  - [Create and deploy CP4BA CR](#create-and-deploy-cp4ba-cr)
  - [Access info after deployment](#access-info-after-deployment-2)
  - [Post-installation](#post-installation)
  - [Customize Decision Center Business console](#customize-decision-center-business-console)
- [Cloud Pak for Business Automation (ODM) Test Environment](#cloud-pak-for-business-automation-test-environment)
  - [Get certified kubernetes folder](#get-certified-kubernetes-folder-1)
  - [Cluster setup](#cluster-setup-1)
  - [Customize CPFS before deployment](#customize-cpfs-before-deployment-1)
  - [Prepare property files](#prepare-property-files-1)
  - [Generate, update and apply SQL and Secret files and validate the connectivity](#generate-update-and-apply-sql-and-secret-files-and-validate-the-connectivity-1)
  - [Create and deploy CP4BA CR](#create-and-deploy-cp4ba-cr-1)
  - [Access info after deployment](#access-info-after-deployment-3)
  - [Post-installation](#post-installation-1)
- [Contacts](#contacts)
- [Notice](#notice)

## Disclaimer ✋

This is **not** an official IBM documentation.  
Absolutely no warranties, no support, no responsibility for anything.  
Use it on your own risk and always follow the official IBM documentations.  
It is always your responsibility to make sure you are license compliant when using this repository to install IBM Cloud Pak for Business Automation.  
Used OpenLDAP and PostgreSQL cannot run on Restricted OCP from CP4BA.

Please do not hesitate to create an issue here if needed. Your feedback is appreciated.

Not for production use. Suitable for Demo and PoC environments - but with Production deployment.

## Prerequisites

- Empty OpenShift cluster of a supported version
- With direct internet connection
- File RWX StorageClass - in this case ocs-storagecluster-cephfs is used, feel free to find and replace
- Block RWO StorageClass - in this case ocs-storagecluster-ceph-rbd is used, feel free to find and replace. File Based RWX is also usable but is not supported.
- Cluster admin user
- IBM entitlement key from https://myibm.ibm.com/products-services/containerlibrary

## Needed tooling

Provided in the install Pod.

- podman
- openjdk9
- oc
- kubectl
- yq
- jq

## Install preparation

In you OCP cluster create Project *cp4ba-install* using OpenShift console.
```yaml
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: cp4ba-install
  labels:
    app: cp4ba-install
```

Create PVC where all files used for installation would be kept if you would need to re-instantiate the install Pod.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: install
  namespace: cp4ba-install
  labels:
    app: cp4ba-install
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
```

Assign cluster admin permissions to *cp4ba-install* default ServiceAccount under which the installation is performed by applying the following yaml.

This requires the logged in OpenShift user to be cluster admin.

The ServiceAccount needs to have cluster admin to be able to create all resources needed to deploy the platform.
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-admin-cp4ba-install
  labels:
    app: cp4ba-install
subjects:
  - kind: User
    apiGroup: rbac.authorization.k8s.io
    name: "system:serviceaccount:cp4ba-install:default"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

Create a pod from which the install will be performed with the following definition.  
It is ready when the message *Install pod - Ready* is in its Log.
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: install
  namespace: cp4ba-install
  labels:
    app: cp4ba-install
spec:
  containers:
    - name: install
      securityContext:
        privileged: true    
      image: ubi9/ubi:9.1.0
      command: ["/bin/bash"]
      args:
        ["-c","cd /usr;
          yum install podman -y;
          yum install ncurses -y;
          yum install jq -y;
          curl -O https://download.java.net/java/GA/jdk9/9/binaries/openjdk-9_linux-x64_bin.tar.gz;
          tar -xvf openjdk-9_linux-x64_bin.tar.gz;
          ln -fs /usr/jdk-9/bin/java /usr/bin/java;
          ln -fs /usr/jdk-9/bin/keytool /usr/bin/keytool;
          curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz --output oc.tar;
          tar -xvf oc.tar oc;
          chmod u+x oc;
          ln -fs /usr/oc /usr/bin/oc;
          curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl;
          chmod u+x kubectl;
          ln -fs /usr/kubectl /usr/bin/kubectl;
          curl -LO https://github.com/mikefarah/yq/releases/download/v4.30.5/yq_linux_amd64.tar.gz;
          tar -xzvf yq_linux_amd64.tar.gz;
          ln -fs /usr/yq_linux_amd64 /usr/bin/yq;
          podman -v;
          java -version;
          keytool;
          oc version;
          kubectl version;
          yq --version;
          jq --version;
          while true;
          do echo 'Install pod - Ready - Enter it via Terminal and \"bash\"';
          sleep 300; done"]      
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: install
          mountPath: /usr/install
  volumes:
    - name: install
      persistentVolumeClaim:
        claimName: install
```

## Command line preparation in install Pod

This needs to be repeated if you don't perform this in one go and you come back to resume.

Open Terminal window of the *install* pod.

Enter bash.
```
bash
```

## Prerequisite software

CP4BA needs at least LDAP and Database. In this deployment OpenLDAP and PostgreSQL are used.

You would normally use your own production ready instances.

Please note that OpenLDAP is working but is not officially supported by CP4BA.

### OpenLDAP

Create cp4ba-openldap Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: cp4ba-openldap
  labels:
    app: cp4ba-openldap
" | oc apply -f -
```

Create ConfigMap for env
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-env
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
data:
  BITNAMI_DEBUG: 'true'
  LDAP_ORGANISATION: cp.internal
  LDAP_ROOT: 'dc=cp,dc=internal'
  LDAP_DOMAIN: cp.internal
" | oc apply -f -
```

Create ConfigMap for users and groups
```bash
echo "
kind: ConfigMap
apiVersion: v1
metadata:
  name: openldap-customldif
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
data:
  01-default-users.ldif: |-
    # cp.internal
    dn: dc=cp,dc=internal
    objectClass: top
    objectClass: dcObject
    objectClass: organization
    o: cp.internal
    dc: cp

    # Units
    dn: ou=Users,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Users

    dn: ou=Groups,dc=cp,dc=internal
    objectClass: organizationalUnit
    ou: Groups

    # Users
    dn: uid=cpadmin,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpadmin
    sn: cpadmin
    uid: cpadmin
    mail: cpadmin@cp.internal
    userpassword: Password
    employeeType: admin

    dn: uid=cpuser,ou=Users,dc=cp,dc=internal
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpuser
    sn: cpuser
    uid: cpuser
    mail: cpuser@cp.internal
    userpassword: Password

    # Groups
    dn: cn=cpadmins,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: top
    cn: cpadmins
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal

    dn: cn=cpusers,ou=Groups,dc=cp,dc=internal
    objectClass: groupOfNames
    objectClass: top
    cn: cpusers
    member: uid=cpadmin,ou=Users,dc=cp,dc=internal
    member: uid=cpuser,ou=Users,dc=cp,dc=internal
" | oc apply -f -    
```

Create Secret for password
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: openldap
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
stringData:
  LDAP_ADMIN_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: openldap-data
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create Deployment
```bash
echo "
kind: Deployment
apiVersion: apps/v1
metadata:
  name: openldap
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cp4ba-openldap
  template:
    metadata:
      labels:
        app: cp4ba-openldap
    spec:
      containers:
        - name: openldap
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          startupProbe:
            tcpSocket:
              port: ldap-port
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 30
          readinessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          livenessProbe:
            tcpSocket:
              port: ldap-port
            initialDelaySeconds: 60
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 10
          terminationMessagePath: /dev/termination-log
          ports:
            - name: ldap-port
              containerPort: 1389
              protocol: TCP
          image: 'bitnami/openldap:2.6.4'
          imagePullPolicy: Always
          volumeMounts:
            - name: data
              mountPath: /bitnami/openldap/
            - name: custom-ldif-files
              mountPath: /ldifs/
          terminationMessagePolicy: File
          envFrom:
            - configMapRef:
                name: openldap-env
            - secretRef:
                name: openldap
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: openldap-data
        - name: custom-ldif-files
          configMap:
            name: openldap-customldif
            defaultMode: 420
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: openldap
  namespace: cp4ba-openldap
  labels:
    app: cp4ba-openldap
spec:
  ports:
    - name: ldap-port
      protocol: TCP
      port: 389
      targetPort: ldap-port
  type: NodePort
  selector:
    app: cp4ba-openldap
" | oc apply -f -
```

Wait for pod in *cp4ba-openldap* Project to become Ready 1/1.
```bash
oc get pod -n cp4ba-openldap -w
```

#### Access info after deployment

OpenLDAP
- See Project cp4ba-openldap, Service openldap for assigned NodePort 
- cn=admin,dc=cp,dc=internal / Password
- In OpenLDAP Pod terminal `ldapsearch -x -b "dc=cp,dc=internal" -H ldap://localhost:1389 -D 'cn=admin,dc=cp,dc=internal' -W "objectclass=*"`

### PostgreSQL

Create cp4ba-postgresql Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: cp4ba-postgresql
  labels:
    app: cp4ba-postgresql
" | oc apply -f -
```

Add required privileged permission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: postgresql-privileged
  namespace: cp4ba-postgresql
  labels:
    app: cp4ba-postgresql
subjects:
  - kind: ServiceAccount
    name: default
    namespace: cp4ba-postgresql
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create Secret for config
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: postgresql-config
  namespace: cp4ba-postgresql
  labels:
    app: cp4ba-postgresql
stringData:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: cpadmin
  POSTGRES_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-data
  namespace: cp4ba-postgresql
  labels:
    app: cp4ba-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create PVC for table spaces as they should not be in PGDATA
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-tablespaces
  namespace: cp4ba-postgresql
  labels:
    app: cp4ba-postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
" | oc apply -f -
```

Create StatefulSet
```bash
echo "
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: cp4ba-postgresql
spec:
  serviceName: postgresql
  replicas: 1
  selector:
    matchLabels:
      app: cp4ba-postgresql
  template:
    metadata:
      labels:
        app: cp4ba-postgresql
    spec:
      containers:
        - name: postgresql
          args:
            - '-c'
            - max_prepared_transactions=500
            - '-c'
            - max_connections=500
          resources:
            limits:
              memory: 4Gi
            requests:
              memory: 4Gi
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 6
            initialDelaySeconds: 5
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 5
          startupProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - exec pg_isready -U \$POSTGRES_USER -d \$POSTGRES_DB
            failureThreshold: 18
            periodSeconds: 10            
          securityContext:
            privileged: true
          image: postgres:14.7-alpine3.17
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgresql-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresql-data
            - mountPath: /pgsqldata
              name: postgresql-tablespaces
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-data
        - name: postgresql-tablespaces
          persistentVolumeClaim:
            claimName: postgresql-tablespaces
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: postgresql
  namespace: cp4ba-postgresql    
  labels:
    app: cp4ba-postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: cp4ba-postgresql
" | oc apply -f -
```

Wait for pod in *cp4ba-postgresql* Project to become Ready 1/1.
```bash
oc get pod -n cp4ba-postgresql -w
```

#### Access info after deployment

PostgreSQL
- See Project cp4ba-postgresql, Service postgresql for assigned NodePort 
- cpadmin / Password
- In PostgreSQL Pod terminal `psql postgresql://cpadmin:Password@localhost:5432/postgresdb`

## Cloud Pak for Business Automation CASE package

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-preparing-client-connect-cluster

```bash
# Download the package
curl https://raw.githubusercontent.com/IBM/cloud-pak/master/repo/case/\
ibm-cp-automation/4.1.4/ibm-cp-automation-4.1.4.tgz \
--output /usr/install/ibm-cp-automation-4.1.4.tgz

# Extract the package
tar xzvf /usr/install/ibm-cp-automation-4.1.4.tgz -C /usr/install
```

## Cloud Pak for Business Automation Development Environment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployments-installing-cp4ba-multi-pattern-production-deployment

### Get certified kubernetes folder

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-preparing-client-connect-cluster
```bash
# Extract cert-kubernetes
tar xvf /usr/install/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-k8s-22.0.2.tar \
-C /usr/install

# Rename directory for dev
mv /usr/install/cert-kubernetes /usr/install/cert-kubernetes-dev
```

### Cluster setup

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=cluster-setting-up-by-running-script

```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-clusteradmin-setup.sh
```

```text
Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 1 # Or 2 based on your platform

What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

Do you want CP4BA Operator support 'All Namespaces'? (Yes/No, default: No) No

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for a new project or an existing project (namespace): cp4ba-dev

The Cloud Pak for Business Automation Operator (Pod, CSV, Subscription) not found in cluster
Continue....

Using project cp4ba-dev...

Here are the existing users on this cluster: 
1) Cluster Admin
2) Your User
Enter an existing username in your cluster, valid option [1 to 3], non-admin is suggested: 2
ATTENTION: When you run cp4a-deployment.sh script, please use cluster admin user.


Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=images-getting-access-from-public-entitled-registry

Do you have a Cloud Pak for Business Automation Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.

Please use available storage classes.

The existing storage classes in the cluster: 
...omitted

Creating docker-registry secret for Entitlement Registry key in project cp4ba-dev...
secret/ibm-entitlement-key created
Done

Waiting for the Cloud Pak for Business Automation operator to be ready. This might take a few minutes... 

catalogsource.operators.coreos.com/ibm-cp4a-operator-catalog created
catalogsource.operators.coreos.com/ibm-cp-automation-foundation-catalog created
catalogsource.operators.coreos.com/ibm-automation-foundation-core-catalog created
catalogsource.operators.coreos.com/opencloud-operators created
catalogsource.operators.coreos.com/bts-operator created
catalogsource.operators.coreos.com/cloud-native-postgresql-catalog created
IBM Operator Catalog source created!
Waiting for CP4A Operator Catalog pod initialization
CP4BA Operator Catalog is running ibm-cp4a-operator-catalog-g624g  1/1   Running  0  26h
operatorgroup.operators.coreos.com/ibm-cp4a-operator-catalog-group created
CP4BA Operator Group Created!
subscription.operators.coreos.com/ibm-cp4a-operator-catalog-subscription created
CP4BA Operator Subscription Created!

Waiting for CP4BA operator pod initialization
No resources found in cp4ba namespace.

CP4A operator is running ibm-cp4a-operator-76c4c85584-pjxrf  1/1   Running   0     17s

Waiting for CP4BA Content operator pod initialization
CP4A Content operator is running ibm-content-operator-58fb9986d7-ccpzk   1/1   Running   0     32s


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed to run the deployment script. For the Starter deployment scenario, you may use one (1) storage class.  For an Production deployment, the deployment script will ask for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of CP4A components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment. 
...omitted list of storage classes
```

Wait until the script finishes.

Wait until all Operators in Project cp4ba-dev are in *Succeeded* state.
```bash
oc get csv -n cp4ba-dev -w
```

### Customize CPFS before deployment

Based on https://www.ibm.com/docs/en/cpfs?topic=services-configuring-foundational-by-using-custom-resource

Change CPFS size to small
```bash
oc patch CommonService common-service --type='merge' -p '{"spec":{"size":"small"}}'
```

Prepare services configurations
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services", "value":[]}]'
```

Change mongodb storage class
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services/-",'\
'"value":{"name":"ibm-mongodb-operator",'\
'"spec":{"mongoDB": {"storageClass":"ocs-storagecluster-ceph-rbd"}}}}]'
```

Customize username of the default CPFS admin
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services/-",'\
'"value":{"name":"ibm-iam-operator",'\
'"spec":{"authentication": {"config":{"defaultAdminUser":"cpfsadmin"}}}}}]'
```

Based on https://www.ibm.com/docs/en/cpfs?topic=operator-versions-compatibility-foundational-services  
Clear SecretShare resource sharing not needed for new versions
```bash
oc patch SecretShare common-services -n cs-control --type=json \
-p '[{"op": "replace", "path": "/spec/configmapshares","value":[]},'\
'{"op": "replace", "path": "/spec/secretshares","value":[]}]'
```

### Prepare property files

Generate properties for components that you would like to deploy.
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-prerequisites.sh -m property
```

Answer as follows.
```text
Tips:Press [ENTER] to accept the default (None of the patterns is selected)
Enter a valid option [1 to 4, 5a, 5b, 6, 7a, 7b]: 2

Hit Enter to continue.

Pattern "Operational Decision Manager": Select optional components: 
1) Decision Center 
2) Rule Execution Server 
3) Decision Runner 
4) Business Automation Insights 

Tips: Decision Center is typically required for development and testing environments. 
Rule Execution Server is typically required for testing and production environments and for using Business Automation Insights. 
You should choose at least one these 2 features to have a minimum environment configuration. 

Tips: Press [ENTER] if you do not want any optional components or when you are finished selecting your optional components
Enter a valid option [1 to 4 or ENTER]: 1

Tips: Press [ENTER] when you are done
Enter a valid option [1 to 4 or ENTER]: 2

Tips: Press [ENTER] when you are done
Enter a valid option [1 to 4 or ENTER]: 3

Pattern "Operational Decision Manager": Select optional components: 
1) Decision Center (Selected)
2) Rule Execution Server (Selected)
3) Decision Runner (Selected)
4) Business Automation Insights 

Tips: Press [ENTER] when you are done
Enter a valid option [1 to 4 or ENTER]:

Hit Enter to continue.

What is the LDAP type used for this deployment? 
1) Microsoft Active Directory
2) Tivoli Directory Server / Security Directory Server
Enter a valid option [1 to 2]: 2
[*] You can change the parameter "LDAP_SSL_ENABLED" in the property file "/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property" later. "LDAP_SSL_ENABLED" is "TRUE" by default.

To provision the persistent volumes and volume claims
please enter the file storage classname for slow storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for medium storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for fast storage(RWX): ocs-storagecluster-cephfs
please enter the block storage classname for Zen(RWO): ocs-storagecluster-ceph-rbd

What is the Database type used for this deployment? 
1) IBM Db2 Database
2) Oracle
3) Microsoft SQL Server
4) PostgreSQL
Enter a valid option [1 to 4]: 4

[*] You can change the parameter "DATABASE_SSL_ENABLE" in the property file "/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "DATABASE_SSL_ENABLE" is "TRUE" by default.
[*] You can change the parameter "POSTGRESQL_SSL_CLIENT_SERVER" in the property file "/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "POSTGRESQL_SSL_CLIENT_SERVER" is "TRUE" by default
[*] - POSTGRESQL_SSL_CLIENT_SERVER="True": For a PostgreSQL database with both server and client authentication
[*] - POSTGRESQL_SSL_CLIENT_SERVER="False": For a PostgreSQL database with server-only authentication

Enter the alias name(s) for the database server(s)/instance(s) to be used by the CP4BA deployment.
(NOTE: NOT the host name of the database server, and CANNOT include a dot[.] character)
(NOTE: This key supports comma-separated lists (for example: dbserver1,dbserver2,dbserver3)
The alias name(s): postgresql

============== Creating database and LDAP property files for CP4BA ==============
Creating DB Server property file for CP4BA
[✔] Created the DB Server property file for CP4BA

Creating LDAP Server property file for CP4BA
[✔] Created the LDAP Server property file for CP4BA


============== Creating property file for database name and user required by CP4BA ==============
Creating Property file for IBM Business Automation Navigator
[✔] Created Property file for IBM Business Automation Navigator

Creating Property file for IBM Operational Decision Manager
[✔] Created Property file for IBM Operational Decision Manager


============== Created all property files for CP4BA ==============
[NEXT ACTIONS]
Enter the <Required> values in the property files under /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile
[*] The key name in the property file is created by the cp4a-prerequisites.sh and is NOT EDITABLE.
[*] The value in the property file must be within double quotes.
[*] The value for User/Password in [cp4ba_db_name_user.property] [cp4ba_user_profile.property] file should NOT include special characters "=" "." "\"
[*] The value in [cp4ba_LDAP.property] or [cp4ba_External_LDAP.property] [cp4ba_user_profile.property] file should NOT include special character '"'
* [cp4ba_db_server.property]:
  - Properties for database server used by CP4BA deployment, such as DATABASE_SERVERNAME/DATABASE_PORT/DATABASE_SSL_ENABLE.

  - The value of "<DB_SERVER_LIST>" is an alias for the database servers. The key supports comma-separated lists.

* [cp4ba_db_name_user.property]:
  - Properties for database name and user name required by each component of the CP4BA deployment, such as GCD_DB_NAME/GCD_DB_USER_NAME/GCD_DB_USER_PASSWORD.

  - Change the prefix "<DB_SERVER_NAME>" to assign which database is used by the component.

  - The value of "<DB_SERVER_NAME>" must match the value of <DB_SERVER_LIST> that is defined in "<DB_SERVER_LIST>" of "cp4ba_db_server.property".

* [cp4ba_LDAP.property]:
  - Properties for the LDAP server that is used by the CP4BA deployment, such as LDAP_SERVER/LDAP_PORT/LDAP_BASE_DN/LDAP_BIND_DN/LDAP_BIND_DN_PASSWORD.

* [cp4ba_user_profile.property]:
  - Properties for the global value used by the CP4BA deployment, such as "sc_deployment_license".

  - properties for the value used by each component of CP4BA, such as <APPLOGIN_USER>/<APPLOGIN_PASSWORD>
```

Update values for cp4ba_db_server.property

```bash
# Backup generated file
cp /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.DATABASE_SERVERNAME="<Required>"/'\
'postgresql.DATABASE_SERVERNAME="postgresql.cp4ba-postgresql.svc.cluster.local"/g' \
-e 's/postgresql.DATABASE_PORT="<Required>"/postgresql.DATABASE_PORT="5432"/g' \
-e 's/postgresql.DATABASE_SSL_ENABLE="True"/postgresql.DATABASE_SSL_ENABLE="False"/g' \
-e 's/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="True"/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="False"/g' \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property
```

Update values for cp4ba_db_name_user.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.ICN_DB_NAME="ICNDB"/postgresql.ICN_DB_NAME="DEVICN"/g' \
-e 's/postgresql.ICN_DB_USER_NAME="<youruser1>"/postgresql.ICN_DB_USER_NAME="devicn"/g' \
-e 's/postgresql.ICN_DB_USER_PASSWORD="<yourpassword>"/postgresql.ICN_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.ODM_DB_NAME="ODMDB"/postgresql.ODM_DB_NAME="DEVODMDB"/g' \
-e 's/postgresql.ODM_DB_USER_NAME="<youruser1>"/postgresql.ODM_DB_USER_NAME="devodm"/g' \
-e 's/postgresql.ODM_DB_USER_PASSWORD="<yourpassword>"/postgresql.ODM_DB_USER_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property
```

Update values for cp4ba_LDAP.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property.bak

# Update generated file with real values
sed -i \
-e 's/LDAP_TYPE="IBM Security Directory Server"/LDAP_TYPE="Custom"/g' \
-e 's/LDAP_SERVER="<Required>"/LDAP_SERVER="openldap.cp4ba-openldap.svc.cluster.local"/g' \
-e 's/LDAP_PORT="<Required>"/LDAP_PORT="389"/g' \
-e 's/LDAP_BASE_DN="<Required>"/LDAP_BASE_DN="dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN="<Required>"/LDAP_BIND_DN="cn=admin,dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN_PASSWORD="<Required>"/LDAP_BIND_DN_PASSWORD="Password"/g' \
-e 's/LDAP_SSL_ENABLED="True"/LDAP_SSL_ENABLED="False"/g' \
-e 's/LDAP_USER_NAME_ATTRIBUTE="<Required>"/LDAP_USER_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_USER_DISPLAY_NAME_ATTR="<Required>"/LDAP_USER_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_BASE_DN="<Required>"/LDAP_GROUP_BASE_DN="ou=Groups,dc=cp,dc=internal"/g' \
-e 's/LDAP_GROUP_NAME_ATTRIBUTE="<Required>"/LDAP_GROUP_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_GROUP_DISPLAY_NAME_ATTR="<Required>"/LDAP_GROUP_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="<Required>"/'\
'LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="'\
'(|(\&(objectclass=groupOfNames)(member={0}))(\&(objectclass=groupofuniquenames)(uniquemember={0})))"/g' \
-e 's/LDAP_GROUP_MEMBER_ID_MAP="<Required>"/LDAP_GROUP_MEMBER_ID_MAP="groupOfNames:member"/g' \
-e 's/LC_USER_FILTER="<Required>"/LC_USER_FILTER="(\&(uid=%v)(objectclass=inetOrgPerson))"/g' \
-e 's/LC_GROUP_FILTER="<Required>"/'\
'LC_GROUP_FILTER="'\
'(\&(cn=%v)(|(objectclass=groupOfNames)(objectclass=groupofuniquenames)(objectclass=groupofurls)))"/g' \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property
```

Update values for cp4ba_user_profile.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property.bak

# Update generated file with real values
sed -i \
-e 's/CP4BA.CP4BA_LICENSE="<Required>"/CP4BA.CP4BA_LICENSE="non-production"/g' \
-e 's/BAN.APPLOGIN_USER="<Required>"/BAN.APPLOGIN_USER="cpadmin"/g' \
-e 's/BAN.APPLOGIN_PASSWORD="<Required>"/BAN.APPLOGIN_PASSWORD="Password"/g' \
-e 's/BAN.LTPA_PASSWORD="<Required>"/BAN.LTPA_PASSWORD="Password"/g' \
-e 's/BAN.KEYSTORE_PASSWORD="<Required>"/BAN.KEYSTORE_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property
```

### Generate, update and apply SQL and Secret files and validate the connectivity

Generate SQL and Secrets
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-prerequisites.sh -m generate

============== Generating DB SQL Statement file required by CP4BA deployment based on property file ==============

Creating the DB SQL statement file for ICN database
[✔] Created the DB SQL statement file for ICN database

Creating the DB SQL statement file for Operational Decision Manager database
[✔] Created the DB SQL statement file for Operational Decision Manager database

[NEXT ACTIONS]
* The DB SQL statement files for CP4BA are created under directory /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/dbscript. You can modify them or use the default setting to create the databases.
(NOTES: DO NOT CHANGE DBNAME/DBUSER/DBPASSWORD DIRECTLY in the DB SQL statement files. CHANGE THEM IN THE PROPERTY FILES IF NEEDED, AND THEN RUN [-m generate] AGAIN)


============== Generating YAML template for secret required by CP4BA deployment based on property file ==============

Creating ldap-bind-secret secret YAML template
[✔] Created ldap-bind-secret secret YAML template

Creating ibm-ban-secret secret YAML template for CP4BA
[✔] Created ibm-ban-secret secret YAML template for CP4BA

Creating Operational Decision Manager secret YAML template
[✔] Created Operational Decision Manager secret YAML template

[NEXT ACTIONS]
* Enter the <Required> values in the YAML templates for the secrets under /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/secret_template

* You can use this shell script to create the secret automatically: /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/create_secret.sh

* Create the databases and Kubernetes secrets manually based on your modified "DB SQL statement file" and "YAML template for secret".
* And then run the  "cp4a-prerequisites.sh -m validate" command to verify that the databases and secrets are created correctly


```
PostgreSQL instance configured in a way that tablespace path in some of the generated SQL create scripts doesn't need updating.

Copy create scripts to PostgreSQL instance
```bash
oc cp /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/dbscript \
cp4ba-postgresql/$(oc get pods --namespace cp4ba-postgresql -o name | cut -d"/" -f2):/usr/dbscript-dev
```

Execute create scripts with table space directory creation
```bash
# Navigator
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/devicn; chown postgres:postgres /pgsqldata/devicn;'
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-dev/ban/postgresql/postgresql/createICNDB.sql'

# ODM
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/devodmdb; chown postgres:postgres /pgsqldata/devodmdb;'
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-dev/odm/postgresql/postgresql/createODMDB.sql'

Nota: Error from server (NotFound): deployments.apps "postgresql" not found

workaround: you could execute sql sentences on postgresl pod directly.
mkdir /pgsqldata/devicn; chown postgres:postgres /pgsqldata/devicn;
psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-dev/ban/postgresql/postgresql/createICNDB.sql
mkdir /pgsqldata/devodmdb; chown postgres:postgres /pgsqldata/devodmdb;
psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-dev/odm/postgresql/postgresql/createODMDB.sql
```

Apply secrets which are in /usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/secret_template/.
```bash
# Switch to Project
oc project cp4ba-dev

# Apply secrets
/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/create_secret.sh
****************************************************************************
******************************* START **************************************
[INFO] Applying YAML template file:/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
secret/ibm-ban-secret created
******************************** END ***************************************
****************************************************************************

****************************************************************************
******************************* START **************************************
[INFO] Applying YAML template file:/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/secret_template/odm/ibm-odm-db-secret.yaml
secret/ibm-odm-db-secret created
******************************** END ***************************************
****************************************************************************

****************************************************************************
******************************* START **************************************
[INFO] Applying YAML template file:/usr/install/cert-kubernetes-dev/scripts/cp4ba-prerequisites/secret_template/ibm-ldap-bind-secret.yaml
secret/ldap-bind-secret created
******************************** END ***************************************
****************************************************************************

```

Validate connectivity
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-prerequisites.sh -m validate
```
Look for all green checkmarks.

### Create and deploy CP4BA CR

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-creating-production

Run deployment script  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-generating-custom-resource-script
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-deployment.sh
```

```text
IMPORTANT: Review the IBM Cloud Pak for Business Automation license information here: 

http://www14.software.ibm.com/cgi-bin/weblap/lap.pl?li_formnum=L-ASAY-CJ9F4D

Press any key to continue

Do you accept the IBM Cloud Pak for Business Automation license (Yes/No, default: No): Yes

Did you deploy Content CR (CRD: contents.icp4a.ibm.com) in current cluster? (Yes/No, default: No): No

Starting to Install the Cloud Pak for Business Automation Operator...


What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

*******************************************************
            Summary of CP4BA capabilities              
*******************************************************
1. Cloud Pak capability to deploy: 
   * Operational Decision Manager
2. Optional components to deploy: 
   * Decision Center
   * Rule Execution Server
   * Decision Runner
*******************************************************

[INFO] Above CP4BA capabilities is already selected in the cp4a-prerequisites.sh script
Press any key to continue


Please select the deployment profile (default: small).  Refer to the documentation in CP4BA Knowledge Center for details on profile.
1) small
2) medium
3) large
Enter a valid option [1 to 3]: 1

Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 1 # Or 2 based on your platform

Provide a URL to zip file that contains JDBC and/or ICCSAP drivers.
(optional - if not provided, the Operator will configure using the default shipped JDBC driver): # Hit Enter - omit

The Operator will configure using the default shipped JDBC driver.

*******************************************************
                    Summary of input                   
*******************************************************
1. Cloud Pak capability to deploy: 
   * Operational Decision Manager
2. Optional components to deploy: 
   * Decision Center
   * Decision Runner
   * Rule Execution Server
3. File storage classname(RWX):
   * Slow: ocs-storagecluster-cephfs
   * Medium: ocs-storagecluster-cephfs
   * Fast: ocs-storagecluster-cephfs
4. Block storage classname(RWO): ocs-storagecluster-ceph-rbd
5. URL to zip file for JDBC and/or ICCSAP drivers: 
*******************************************************

Verify that the information above is correct.
To proceed with the deployment, enter "Yes".
To make changes, enter "No" (default: No): Yes

Creating the Custom Resource of the Cloud Pak for Business Automation operator...

Applying value in property file into final CR
[✔] Applied value in property file into final CR under /usr/install/cert-kubernetes-dev/scripts/generated-cr
Please confirm final custom resource under /usr/install/cert-kubernetes-dev/scripts/generated-cr

The custom resource file used is: "/usr/install/cert-kubernetes-dev/scripts/generated-cr/ibm_cp4a_cr_final.yaml"

ATTENTION: If the cluster is running a Linux on Z (s390x)/Power architecture, remove the baml_configuration section from "/usr/install/cert-kubernetes-dev/scripts/generated-cr/ibm_cp4a_cr_final.yaml" before you apply the custom resource. Business Automation Machine Learning Server (BAML) is not supported on this architecture.


To monitor the deployment status, follow the Operator logs.
For details, refer to the troubleshooting section in Knowledge Center here: 
https://www.ibm.com/support/knowledgecenter/SSYHZ8_22.0.2/com.ibm.dba.install/op_topics/tsk_trbleshoot_operators.html
```

Update CR file  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-checking-completing-your-custom-resource

Change LDAP sub settings from tds to custom
```bash
sed -i \
-e 's/tds:/custom:/g' \
/usr/install/cert-kubernetes-dev/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Apply CR  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-deploying-custom-resource-you-created-script
```bash
oc apply -n cp4ba-dev -f /usr/install/cert-kubernetes-dev/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Wait for the deployment to be completed. Can be determined by looking in Project cp4ba-dev in Kind ICP4ACluster, instance named icp4adeploy to have the following conditions:
```bash
oc get -n cp4ba-dev icp4acluster icp4adeploy -o=jsonpath="{.status.conditions}" -w
```
```yaml
  conditions:
    - message: Running reconciliation
      reason: Running
      status: 'True'
      type: Running
    - message: Prerequisites execution done.
      reason: Successful
      status: 'True'
      type: PrereqReady
    - message: ''
      reason: Successful
      status: 'True'
      type: Ready
```

### Access info after deployment

- https://cpd-cp4ba-dev.<ocp_apps_domain>/
- cpadmin / Password
- Additional capabilities based info in Project cp4ba-dev in ConfigMap icp4adeploy-cp4ba-access-info


### Post Installation
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=tasks-operational-decision-manager
You need to setup permissions for your users.  
Before that you need to add cpfsadmin user to Zen to be able to follow step 3 in the Docs.

<img width="1092" alt="image" src="https://user-images.githubusercontent.com/83011523/237059072-1bbfe22a-cd36-4bd4-b431-1861b40e111c.png">


```bash
# Get password of zen initial admin
zen_admin_password=`oc get secret admin-user-details -n cp4ba-dev \
-o jsonpath='{.data.initial_admin_password}' | base64 -d`
echo $zen_admin_password

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n cp4ba-dev \
-o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get zen token
# Based on https://cloud.ibm.com/apidocs/cloud-pak-data/cloud-pak-data-4.5.0#getauthorizationtoken
zen_token=`curl --silent -k -H "Content-Type: application/json" \
-d '{"username":"admin","password":"'$zen_admin_password'"}' \
https://cpd-cp4ba-dev.${apps_endpoint}/icp4d-api/v1/authorize | jq -r ".token"`
echo $zen_token

# Add cpfsadmin user to zen as administrator
curl -kv -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpfsadmin","displayName":"cpfsadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst","iaf-automation-developer",'\
'"iaf-automation-operator","zen_user_role","odm_admin_role","odm_business_user_role","odm_runtime_admin_role","odm_runtime_user_role"]}' \
https://cpd-cp4ba-dev.${apps_endpoint}/usermgmt/v1/user

# To get cpfsadmin password
oc get secret platform-auth-idp-credentials -n cp4ba-dev -o jsonpath='{.data.admin_password}' | base64 -d
```

Also add all roles to cpadmin user to be usable for all tasks.

```bash
curl -kv -X PUT -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst",'\
'"iaf-automation-developer","iaf-automation-operator","zen_user_role","odm_admin_role","odm_business_user_role","odm_runtime_admin_role","odm_runtime_user_role"]}' \
https://cpd-cp4ba-dev.${apps_endpoint}/usermgmt/v1/user/cpadmin?add_roles=true
```

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=access-managing-user-permissions

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-recommended-validating-your-production you can further verify the environemnts and get important information. But before running anything else then --help follow additional steps for script configuration.
```bash
oc project cp4ba-dev
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --help

 
##########################################################################################################
                              Running CP4BA Post install  Service
##########################################################################################################

Help Menu for service flags
################################################################
--Precheck                           This will precheck OCP access
--StarterStatus                      This will give status for services for Starter
--StarterConsole                     This will give console for services URLs for Starter
--StarterProbe                       This will probe readiness/liveness of the Starter endpoints
--Status                             This will give status for services for Production
--Console                            This will give console for services URLs for Production
--Probe                              This will probe readiness/liveness of the Production endpoints
--RPAStatus                          This will give status for RPA Server
--RPAConsole                         This will give console for RPA Server

##########################################################################################################
```

Perform setup for cp4a-post-install.sh script
```bash
# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n cp4ba-dev -o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get CPFS token
cpfs_token=`curl --silent -k --header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'username=cpadmin' \
--data-urlencode 'password=Password' \
--data-urlencode 'scope=openid' \
https://cp-console-cp4ba-dev.${apps_endpoint}/idprovider/v1/auth/identitytoken | jq -r ".access_token"`
echo $cpfs_token

# Exchange CPFS token for Zen token
zen_token=`curl --silent -k -H "Content-Type: application/json" \
--header "iam-token: ${cpfs_token}" \
--header 'username: cpadmin' \
https://cpd-cp4ba-dev.${apps_endpoint}/v1/preauth/validateAuth | jq -r ".accessToken"`
echo $zen_token

# Generate ZenApiKey
zen_api_key=`curl --silent -k -H "Content-Type: application/json" \
--header "Authorization: Bearer ${zen_token}" \
--header 'username: cpadmin' \
https://cpd-cp4ba-dev.${apps_endpoint}/usermgmt/v1/user/apiKey | jq -r ".apiKey"`
echo $zen_api_key

# Change CPFS namespace for cp4a-post-install.sh script and add password and zen api key
sed -i \
-e 's/CP4BA_COMMON_SERVICES_NAMESPACE="ibm-common-services"/'\
'CP4BA_COMMON_SERVICES_NAMESPACE="cp4ba-dev"/g' \
-e 's/PROBE_USER_API_KEY=/'\
'PROBE_USER_API_KEY="'${zen_api_key}'"/g' \
-e 's/PROBE_USER_NAME=/'\
'PROBE_USER_NAME="cpadmin"/g' \
-e 's/PROBE_USER_PASSWORD=/'\
'PROBE_USER_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-dev/scripts/helper/post-install/env.sh
```

Now you can run cp4a-post-install.sh with other parameters.
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --precheck
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --status
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --console
```

Follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-completing-post-installation-tasks as needed.

Custom CPFS console TLS - follow https://www.ibm.com/docs/en/cloud-paks/1.0?topic=management-replacing-foundational-services-endpoint-certificates#rep_cs370

Custom CPFS admin password - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=tasks-cloud-pak-foundational-services

Custom Zen certificates - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=security-customizing-cloud-pak-entry-point

### Customize Decision Center Business console

The Decision Center Business console can be customized with your own dynamic domains, value editors, or ruleset extractors.

To be able to add your customizations to the Decision Center Business console, you must first create a persistent volume claim (PVC) dedicated to the storage of the custom libraries.

```yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:  
    name: custom-dc-libs-pvc
  spec:
    accessModes:    
      - ReadWriteOnce  
    resources:    
      requests:      
        storage: 1Gi
    storageClassName: ocs-storagecluster-cephfs    
```
You must also set the parameter decisionCenter.customlibPvc to point to your PVC.

<img width="970" alt="image" src="https://github.com/drodriguezflores/cp4ba-odm-manual-deployment/assets/83011523/aecb8bb5-cf4f-4ff6-b7ce-fd36e7feb30a">

Once, the icp4adeploy reconciliation finish you could validate this configuration. 

<img width="1103" alt="image" src="https://github.com/drodriguezflores/cp4ba-odm-manual-deployment/assets/83011523/0123ddb5-a210-4273-93c8-fe04ef6ae363">

Then, add the Decision Center custom library to the Decision Center pod with the following command.

```bash
oc cp mycustomdclib.jar namespace/my-odm-prod-instance-odm-decisioncenter-XXXXXX:/config/customlib
```
Make sure that the JAR files are in the running pod.

```bash
oc exec my-odm-prod-instance-odm-decisioncenter-XXXXXX -- ls -l /config/customlib
```

References:
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=manager-configuring-decision-center-business-console

https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=manager-customizing-decision-center-business-console


## Cloud Pak for Business Automation Test Environment

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployments-installing-cp4ba-multi-pattern-production-deployment

### Get certified kubernetes folder

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-preparing-client-connect-cluster
```bash
# Extract cert-kubernetes
tar xvf /usr/install/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-k8s-22.0.2.tar \
-C /usr/install

# Rename directory for test
mv /usr/install/cert-kubernetes /usr/install/cert-kubernetes-test
```

### Cluster setup

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=cluster-setting-up-by-running-script

```bash
/usr/install/cert-kubernetes-test/scripts/cp4a-clusteradmin-setup.sh
```

```text
Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 1 # Or 2 based on your platform

What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

Do you want CP4BA Operator support 'All Namespaces'? (Yes/No, default: No) No

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for a new project or an existing project (namespace): cp4ba-test

Found the existing Cloud Pak for Business Automation Operator (Pod, CSV, Subscription) in different project "cp4ba-dev"! 

Do you want to deploy another CP4BA Operator in new project "cp4ba-test"? (Yes/No, default: No) Yes
Continue....

true
Using project cp4ba-test...

Here are the existing users on this cluster: 
1) Cluster Admin
2) Your User
Enter an existing username in your cluster, valid option [1 to 3], non-admin is suggested: 2
ATTENTION: When you run cp4a-deployment.sh script, please use cluster admin user.


Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=images-getting-access-from-public-entitled-registry

Do you have a Cloud Pak for Business Automation Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.

Please use available storage classes.

The existing storage classes in the cluster: 
...omitted

Creating docker-registry secret for Entitlement Registry key in project cp4ba-test...
secret/ibm-entitlement-key created
Done

Waiting for the Cloud Pak for Business Automation operator to be ready. This might take a few minutes... 

catalogsource.operators.coreos.com/ibm-cp4a-operator-catalog created
catalogsource.operators.coreos.com/ibm-cp-automation-foundation-catalog created
catalogsource.operators.coreos.com/ibm-automation-foundation-core-catalog created
catalogsource.operators.coreos.com/opencloud-operators created
catalogsource.operators.coreos.com/bts-operator created
catalogsource.operators.coreos.com/cloud-native-postgresql-catalog created
IBM Operator Catalog source created!
Waiting for CP4A Operator Catalog pod initialization
CP4BA Operator Catalog is running ibm-cp4a-operator-catalog-g624g  1/1   Running  0  26h
operatorgroup.operators.coreos.com/ibm-cp4a-operator-catalog-group created
CP4BA Operator Group Created!
subscription.operators.coreos.com/ibm-cp4a-operator-catalog-subscription created
CP4BA Operator Subscription Created!

Waiting for CP4BA operator pod initialization
No resources found in cp4ba namespace.

CP4A operator is running ibm-cp4a-operator-76c4c85584-pjxrf  1/1   Running   0     17s

Waiting for CP4BA Content operator pod initialization
CP4A Content operator is running ibm-content-operator-58fb9986d7-ccpzk   1/1   Running   0     32s


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed to run the deployment script. For the Starter deployment scenario, you may use one (1) storage class.  For an Production deployment, the deployment script will ask for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of CP4A components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment. 
...omitted list of storage classes
```

Wait until the script finishes.

Wait until all Operators in Project cp4ba-test are in *Succeeded* state.
```bash
oc get csv -n cp4ba-test -w
```

### Customize CPFS before deployment

Based on https://www.ibm.com/docs/en/cpfs?topic=services-configuring-foundational-by-using-custom-resource

Change CPFS size to small
```bash
oc patch CommonService common-service --type='merge' -p '{"spec":{"size":"small"}}'
```

Prepare services configurations
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services", "value":[]}]'
```

Change mongodb storage class
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services/-",'\
'"value":{"name":"ibm-mongodb-operator",'\
'"spec":{"mongoDB": {"storageClass":"ocs-storagecluster-ceph-rbd"}}}}]'
```

Customize username of the default CPFS admin
```bash
oc patch CommonService common-service --type=json \
-p '[{"op": "add", "path": "/spec/services/-",'\
'"value":{"name":"ibm-iam-operator",'\
'"spec":{"authentication": {"config":{"defaultAdminUser":"cpfsadmin"}}}}}]'
```

### Prepare property files

Generate properties for components that you would like to deploy.
```bash
/usr/install/cert-kubernetes-test/scripts/cp4a-prerequisites.sh -m property
```

Answer as follows.
```text
Tips:Press [ENTER] to accept the default (None of the patterns is selected)
Enter a valid option [1 to 4, 5a, 5b, 6, 7a, 7b]: 2

Tips:Press [ENTER] to accept the default (None of the patterns is selected)
Enter a valid option [1 to 4, 5a, 5b, 6, 7a, 7b]: 

Hit Enter to continue.

Pattern "Operational Decision Manager": Select optional components: 
1) Decision Center (Selected)
2) Rule Execution Server (Selected)
3) Decision Runner 
4) Business Automation Insights 

Tips: Decision Center is typically required for development and testing environments. 
Rule Execution Server is typically required for testing and production environments and for using Business Automation Insights. 
You should choose at least one these 2 features to have a minimum environment configuration. 

Hit Enter to continue.

Tips: Press [ENTER] to accept the default (None of the components is selected)
Enter a valid option [1 to 1 or ENTER]: 1

Enter a valid option [1 to 1 or ENTER]: 2

Hit Enter to continue.

What is the LDAP type used for this deployment? 
1) Microsoft Active Directory
2) Tivoli Directory Server / Security Directory Server
Enter a valid option [1 to 2]: 2
[*] You can change the parameter "LDAP_SSL_ENABLED" in the property file "/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property" later. "LDAP_SSL_ENABLED" is "TRUE" by default.

To provision the persistent volumes and volume claims
please enter the file storage classname for slow storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for medium storage(RWX): ocs-storagecluster-cephfs
please enter the file storage classname for fast storage(RWX): ocs-storagecluster-cephfs
please enter the block storage classname for Zen(RWO): ocs-storagecluster-ceph-rbd

What is the Database type used for this deployment? 
1) IBM Db2 Database
2) Oracle
3) Microsoft SQL Server
4) PostgreSQL
Enter a valid option [1 to 4]: 4

[*] You can change the parameter "DATABASE_SSL_ENABLE" in the property file "/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "DATABASE_SSL_ENABLE" is "TRUE" by default.
[*] You can change the parameter "POSTGRESQL_SSL_CLIENT_SERVER" in the property file "/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property" later. "POSTGRESQL_SSL_CLIENT_SERVER" is "TRUE" by default
[*] - POSTGRESQL_SSL_CLIENT_SERVER="True": For a PostgreSQL database with both server and client authentication
[*] - POSTGRESQL_SSL_CLIENT_SERVER="False": For a PostgreSQL database with server-only authentication

Enter the alias name(s) for the database server(s)/instance(s) to be used by the CP4BA deployment.
(NOTE: NOT the host name of the database server, and CANNOT include a dot[.] character)
(NOTE: This key supports comma-separated lists (for example: dbserver1,dbserver2,dbserver3)
The alias name(s): postgresql

============== Creating database and LDAP property files for CP4BA ==============
Creating DB Server property file for CP4BA
[✔] Created the DB Server property file for CP4BA

Creating LDAP Server property file for CP4BA
[✔] Created the LDAP Server property file for CP4BA


============== Creating property file for database name and user required by CP4BA ==============
Creating Property file for IBM Business Automation Navigator
[✔] Created Property file for IBM Business Automation Navigator

Creating Property file for IBM Operational Decision Manager
[✔] Created Property file for IBM Operational Decision Manager


============== Created all property files for CP4BA ==============
[NEXT ACTIONS]
Enter the <Required> values in the property files under /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile
[*] The key name in the property file is created by the cp4a-prerequisites.sh and is NOT EDITABLE.
[*] The value in the property file must be within double quotes.
[*] The value for User/Password in [cp4ba_db_name_user.property] [cp4ba_user_profile.property] file should NOT include special characters "=" "." "\"
[*] The value in [cp4ba_LDAP.property] or [cp4ba_External_LDAP.property] [cp4ba_user_profile.property] file should NOT include special character '"'
* [cp4ba_db_server.property]:
  - Properties for database server used by CP4BA deployment, such as DATABASE_SERVERNAME/DATABASE_PORT/DATABASE_SSL_ENABLE.

  - The value of "<DB_SERVER_LIST>" is an alias for the database servers. The key supports comma-separated lists.

* [cp4ba_db_name_user.property]:
  - Properties for database name and user name required by each component of the CP4BA deployment, such as GCD_DB_NAME/GCD_DB_USER_NAME/GCD_DB_USER_PASSWORD.

  - Change the prefix "<DB_SERVER_NAME>" to assign which database is used by the component.

  - The value of "<DB_SERVER_NAME>" must match the value of <DB_SERVER_LIST> that is defined in "<DB_SERVER_LIST>" of "cp4ba_db_server.property".

* [cp4ba_LDAP.property]:
  - Properties for the LDAP server that is used by the CP4BA deployment, such as LDAP_SERVER/LDAP_PORT/LDAP_BASE_DN/LDAP_BIND_DN/LDAP_BIND_DN_PASSWORD.

* [cp4ba_user_profile.property]:
  - Properties for the global value used by the CP4BA deployment, such as "sc_deployment_license".

  - properties for the value used by each component of CP4BA, such as <APPLOGIN_USER>/<APPLOGIN_PASSWORD>
```

Update values for cp4ba_db_server.property

```bash
# Backup generated file
cp /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.DATABASE_SERVERNAME="<Required>"/'\
'postgresql.DATABASE_SERVERNAME="postgresql.cp4ba-postgresql.svc.cluster.local"/g' \
-e 's/postgresql.DATABASE_PORT="<Required>"/postgresql.DATABASE_PORT="5432"/g' \
-e 's/postgresql.DATABASE_SSL_ENABLE="True"/postgresql.DATABASE_SSL_ENABLE="False"/g' \
-e 's/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="True"/postgresql.POSTGRESQL_SSL_CLIENT_SERVER="False"/g' \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property
```

Update values for cp4ba_db_name_user.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property.bak
```
```bash
# Update generated file with real values
sed -i \
-e 's/postgresql.ICN_DB_NAME="ICNDB"/postgresql.ICN_DB_NAME="TESTICN"/g' \
-e 's/postgresql.ICN_DB_USER_NAME="<youruser1>"/postgresql.ICN_DB_USER_NAME="testicn"/g' \
-e 's/postgresql.ICN_DB_USER_PASSWORD="<yourpassword>"/postgresql.ICN_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.ODM_DB_NAME="ODMDB"/postgresql.ODM_DB_NAME="TESTODMDB"/g' \
-e 's/postgresql.ODM_DB_USER_NAME="<youruser1>"/postgresql.ODM_DB_USER_NAME="testodm"/g' \
-e 's/postgresql.ODM_DB_USER_PASSWORD="<yourpassword>"/'\
'postgresql.ODM_DB_USER_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property
```

Update values for cp4ba_LDAP.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property.bak

# Update generated file with real values
sed -i \
-e 's/LDAP_TYPE="IBM Security Directory Server"/LDAP_TYPE="Custom"/g' \
-e 's/LDAP_SERVER="<Required>"/LDAP_SERVER="openldap.cp4ba-openldap.svc.cluster.local"/g' \
-e 's/LDAP_PORT="<Required>"/LDAP_PORT="389"/g' \
-e 's/LDAP_BASE_DN="<Required>"/LDAP_BASE_DN="dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN="<Required>"/LDAP_BIND_DN="cn=admin,dc=cp,dc=internal"/g' \
-e 's/LDAP_BIND_DN_PASSWORD="<Required>"/LDAP_BIND_DN_PASSWORD="Password"/g' \
-e 's/LDAP_SSL_ENABLED="True"/LDAP_SSL_ENABLED="False"/g' \
-e 's/LDAP_USER_NAME_ATTRIBUTE="<Required>"/LDAP_USER_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_USER_DISPLAY_NAME_ATTR="<Required>"/LDAP_USER_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_BASE_DN="<Required>"/LDAP_GROUP_BASE_DN="ou=Groups,dc=cp,dc=internal"/g' \
-e 's/LDAP_GROUP_NAME_ATTRIBUTE="<Required>"/LDAP_GROUP_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_GROUP_DISPLAY_NAME_ATTR="<Required>"/LDAP_GROUP_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="<Required>"/'\
'LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="'\
'(|(\&(objectclass=groupOfNames)(member={0}))(\&(objectclass=groupofuniquenames)(uniquemember={0})))"/g' \
-e 's/LDAP_GROUP_MEMBER_ID_MAP="<Required>"/LDAP_GROUP_MEMBER_ID_MAP="groupOfNames:member"/g' \
-e 's/LC_USER_FILTER="<Required>"/LC_USER_FILTER="(\&(uid=%v)(objectclass=inetOrgPerson))"/g' \
-e 's/LC_GROUP_FILTER="<Required>"/'\
'LC_GROUP_FILTER="'\
'(\&(cn=%v)(|(objectclass=groupOfNames)(objectclass=groupofuniquenames)(objectclass=groupofurls)))"/g' \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property
```

Update values for cp4ba_user_profile.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property.bak

# Update generated file with real values
sed -i \
-e 's/CP4BA.CP4BA_LICENSE="<Required>"/CP4BA.CP4BA_LICENSE="non-production"/g' \
-e 's/BAN.APPLOGIN_USER="<Required>"/BAN.APPLOGIN_USER="cpadmin"/g' \
-e 's/BAN.APPLOGIN_PASSWORD="<Required>"/BAN.APPLOGIN_PASSWORD="Password"/g' \
-e 's/BAN.LTPA_PASSWORD="<Required>"/BAN.LTPA_PASSWORD="Password"/g' \
-e 's/BAN.KEYSTORE_PASSWORD="<Required>"/BAN.KEYSTORE_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/propertyfile/cp4ba_user_profile.property
```

### Generate, update and apply SQL and Secret files and validate the connectivity

Generate SQL and Secrets
```bash
/usr/install/cert-kubernetes-test/scripts/cp4a-prerequisites.sh -m generate
```
PostgreSQL instance configured in a way that tablespace path in some of the generated SQL create scripts doesn't need updating.

Copy create scripts to PostgreSQL instance
```bash
oc cp /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/dbscript \
cp4ba-postgresql/$(oc get pods --namespace cp4ba-postgresql -o name | cut -d"/" -f2):/usr/dbscript-test
```

Execute create scripts with table space directory creation
```bash
# Navigator
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/testicn; chown postgres:postgres /pgsqldata/testicn;'
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-test/ban/postgresql/postgresql/createICNDB.sql'

# ODM
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/testodmdb; chown postgres:postgres /pgsqldata/testodmdb;'
oc --namespace cp4ba-postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
--file=/usr/dbscript-test/odm/postgresql/postgresql/createODMDB.sql'

```

Apply secrets which are in /usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/secret_template/.
```bash
# Switch to Project
oc project cp4ba-test

# Apply secrets
/usr/install/cert-kubernetes-test/scripts/cp4ba-prerequisites/create_secret.sh
```

Validate connectivity
```bash
/usr/install/cert-kubernetes-test/scripts/cp4a-prerequisites.sh -m validate
```
Look for all green checkmarks.

### Create and deploy CP4BA CR

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-creating-production

Run deployment script  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-generating-custom-resource-script
```bash
/usr/install/cert-kubernetes-test/scripts/cp4a-deployment.sh
```

```text
IMPORTANT: Review the IBM Cloud Pak for Business Automation license information here: 

http://www14.software.ibm.com/cgi-bin/weblap/lap.pl?li_formnum=L-ASAY-CCQCLE

Press any key to continue

Do you accept the IBM Cloud Pak for Business Automation license (Yes/No, default: No): Yes

Did you deploy Content CR (CRD: contents.icp4a.ibm.com) in current cluster? (Yes/No, default: No): No

Starting to Install the Cloud Pak for Business Automation Operator...


What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

*******************************************************
            Summary of CP4BA capabilities              
*******************************************************
1. Cloud Pak capability to deploy: 
   * Operational Decision Manager
2. Optional components to deploy: 
   * Decision Center
   * Rule Execution Server
*******************************************************

[INFO] Above CP4BA capabilities is already selected in the cp4a-prerequisites.sh script
Press any key to continue


Please select the deployment profile (default: small).  Refer to the documentation in CP4BA Knowledge Center for details on profile.
1) small
2) medium
3) large
Enter a valid option [1 to 3]: 1

Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 1 # Or 2 based on your platform

Provide a URL to zip file that contains JDBC and/or ICCSAP drivers.
(optional - if not provided, the Operator will configure using the default shipped JDBC driver): # Hit Enter - omit

The Operator will configure using the default shipped JDBC driver.

*******************************************************
                    Summary of input                   
*******************************************************
1. Cloud Pak capability to deploy: 
   * Operational Decision Manager
2. Optional components to deploy: 
   * Decision Center
   * Rule Execution Server
3. File storage classname(RWX):
   * Slow: ocs-storagecluster-cephfs
   * Medium: ocs-storagecluster-cephfs
   * Fast: ocs-storagecluster-cephfs
4. Block storage classname(RWO): ocs-storagecluster-ceph-rbd
5. URL to zip file for JDBC and/or ICCSAP drivers: 
*******************************************************

Verify that the information above is correct.
To proceed with the deployment, enter "Yes".
To make changes, enter "No" (default: No): Yes

Creating the Custom Resource of the Cloud Pak for Business Automation operator...

Applying value in property file into final CR
[✔] Applied value in property file into final CR under /usr/install/cert-kubernetes-test/scripts/generated-cr
Please confirm final custom resource under /usr/install/cert-kubernetes-test/scripts/generated-cr

The custom resource file used is: "/usr/install/cert-kubernetes-test/scripts/generated-cr/ibm_cp4a_cr_final.yaml"

ATTENTION: If the cluster is running a Linux on Z (s390x)/Power architecture, remove the baml_configuration section from "/usr/install/cert-kubernetes-test/scripts/generated-cr/ibm_cp4a_cr_final.yaml" before you apply the custom resource. Business Automation Machine Learning Server (BAML) is not supported on this architecture.


To monitor the deployment status, follow the Operator logs.
For details, refer to the troubleshooting section in Knowledge Center here: 
https://www.ibm.com/support/knowledgecenter/SSYHZ8_22.0.2/com.ibm.dba.install/op_topics/tsk_trbleshoot_operators.html
```

Update CR file  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-checking-completing-your-custom-resource

Change LDAP sub settings from tds to custom
```bash
sed -i \
-e 's/tds:/custom:/g' \
/usr/install/cert-kubernetes-test/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Apply CR  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-deploying-custom-resource-you-created-script
```bash
oc apply -n cp4ba-test -f /usr/install/cert-kubernetes-test/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Wait for the deployment to be completed. Can be determined by looking in Project cp4ba-test in Kind ICP4ACluster, instance named icp4adeploy to have the following conditions:
```bash
oc get -n cp4ba-test icp4acluster icp4adeploy -o=jsonpath="{.status.conditions}" -w
```
```yaml
  conditions:
    - message: Running reconciliation
      reason: Running
      status: 'True'
      type: Running
    - message: Prerequisites execution done.
      reason: Successful
      status: 'True'
      type: PrereqReady
    - message: ''
      reason: Successful
      status: 'True'
      type: Ready
```

### Access info after deployment

- https://cpd-cp4ba-test.<ocp_apps_domain>/
- cpadmin / Password
- Additional capabilities based info in Project cp4ba-test in ConfigMap icp4adeploy-cp4ba-access-info

### Post-installation

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-recommended-validating-your-production you can further verify the environemnts and get important information.
```bash
oc project cp4ba-test
/usr/install/cert-kubernetes-test/scripts/cp4a-post-install.sh --help
```

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=cpbaf-business-automation-studio  
You need to setup permissions for your users.  

<img width="1752" alt="image" src="https://user-images.githubusercontent.com/83011523/237045700-154112a9-9851-4910-b24a-80b95eb56e65.png">

Before that you need to add cpfsadmin user to Zen to be able to follow step 3 in the Docs.
```bash
# Get password of zen initial admin
zen_admin_password=`oc get secret admin-user-details -n cp4ba-test \
-o jsonpath='{.data.initial_admin_password}' | base64 -d`
echo $zen_admin_password

# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n cp4ba-test \
-o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get zen token
# Based on https://cloud.ibm.com/apidocs/cloud-pak-data/cloud-pak-data-4.5.0#getauthorizationtoken
zen_token=`curl --silent -k -H "Content-Type: application/json" \
-d '{"username":"admin","password":"'$zen_admin_password'"}' \
https://cpd-cp4ba-test.${apps_endpoint}/icp4d-api/v1/authorize | jq -r ".token"`
echo $zen_token

# Add cpfsadmin user to zen as administrator
curl -kv -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpfsadmin","displayName":"cpfsadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst",'\
'"iaf-automation-developer","iaf-automation-operator","zen_user_role","odm_admin_role","odm_business_user_role","odm_runtime_admin_role","odm_runtime_user_role"]}' \
https://cpd-cp4ba-test.${apps_endpoint}/usermgmt/v1/user

# To get cpfsadmin password
oc get secret platform-auth-idp-credentials -n cp4ba-test -o jsonpath='{.data.admin_password}' | base64 -d
```

Also add all roles to cpadmin user to be usable for all tasks.

```bash
curl -kv -X PUT -H "Content-Type: application/json" -H "Authorization: Bearer $zen_token" \
-d '{"username":"cpadmin", '\
'"user_roles":["zen_administrator_role","iaf-automation-admin","iaf-automation-analyst",'\
'"iaf-automation-developer","iaf-automation-operator","zen_user_role","odm_admin_role","odm_business_user_role","odm_runtime_admin_role","odm_runtime_user_role"]}' \
https://cpd-cp4ba-test.${apps_endpoint}/usermgmt/v1/user/cpadmin?add_roles=true
```

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-recommended-validating-your-production you can further verify the environemnts and get important information. But before running anything else then --help follow additional steps for script configuration.
```bash
oc project cp4ba-test
/usr/install/cert-kubernetes-test/scripts/cp4a-post-install.sh --help
```

Perform setup for cp4a-post-install.sh script
```bash
# Get apps endpoint of your openshift
apps_endpoint=`oc get ingress.v1.config.openshift.io cluster -n cp4ba-dev -o jsonpath='{.spec.domain}'`
echo $apps_endpoint

# Get CPFS token
cpfs_token=`curl --silent -k --header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'username=cpadmin' \
--data-urlencode 'password=Password' \
--data-urlencode 'scope=openid' \
https://cp-console-cp4ba-test.${apps_endpoint}/idprovider/v1/auth/identitytoken | jq -r ".access_token"`
echo $cpfs_token

# Exchange CPFS token for Zen token
zen_token=`curl --silent -k -H "Content-Type: application/json" \
--header "iam-token: ${cpfs_token}" \
--header 'username: cpadmin' \
https://cpd-cp4ba-test.${apps_endpoint}/v1/preauth/validateAuth | jq -r ".accessToken"`
echo $zen_token

# Generate ZenApiKey
zen_api_key=`curl --silent -k -H "Content-Type: application/json" \
--header "Authorization: Bearer ${zen_token}" \
--header 'username: cpadmin' \
https://cpd-cp4ba-test.${apps_endpoint}/usermgmt/v1/user/apiKey | jq -r ".apiKey"`
echo $zen_api_key

# Change CPFS namespace for cp4a-post-install.sh script and add password and zen api key
sed -i \
-e 's/CP4BA_COMMON_SERVICES_NAMESPACE="ibm-common-services"/'\
'CP4BA_COMMON_SERVICES_NAMESPACE="cp4ba-dev"/g' \
-e 's/PROBE_USER_API_KEY=/'\
'PROBE_USER_API_KEY="'${zen_api_key}'"/g' \
-e 's/PROBE_USER_NAME=/'\
'PROBE_USER_NAME="cpadmin"/g' \
-e 's/PROBE_USER_PASSWORD=/'\
'PROBE_USER_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes-test/scripts/helper/post-install/env.sh
```

Now you can run cp4a-post-install.sh with other parameters.
```bash
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --precheck
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --status
/usr/install/cert-kubernetes-dev/scripts/cp4a-post-install.sh --console
```

Follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=deployment-completing-post-installation-tasks as needed.

Custom CPFS console TLS - follow https://www.ibm.com/docs/en/cloud-paks/1.0?topic=management-replacing-foundational-services-endpoint-certificates#rep_cs370

Custom Zen certificates - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=security-customizing-cloud-pak-entry-point

Custom CPFS admin password - follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.2?topic=tasks-cloud-pak-foundational-services


## Contacts

Diego Rodriguez Flores

Juan Diego Garcia Hernandez

Customer Success Manager

IBM SPGI

## Notice

© Copyright IBM Corporation 2023.
