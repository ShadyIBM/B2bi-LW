# Receipe for deploying b2bi 6.2 and lightwell framework

## Table of Contents

- [Create-the-Cluster](#Create the Cluster)
- [Deploying-B2BI](#Deploying the B2BI Sterling File Gateway in a single namespace)
- [LightWell-Deployment](#LightWell Deployment)
- [Post-installation-Config](#Post installation Config)
  
## Create the Cluster
- Provision a Red Hat OpenShift on IBM Cloud cluster with GitOps Configuration from IBM Technology Zone. Select the ROKS Cluster with GitOps Configuration tile.

- Click the Reserve now radio button.
- Provide a name for the cluster, make a selection for the purpose and choose the region to provision the cluster.
- Once a Preferred Geography has been selected, provide the appropriate Worker Node Count and Worker Node Flavor values based on the requirements for your cluster. Finally, click Submit

> Note: The default selected configurations should be sufficient if you are using the cluster for training or practice.

### Install required CLIs
- I found the easier way to install the CLIs is to use Homebrew
- Once you have installed Homebrew, install the Github CLI
- Then install Openshift CLI, and the Kubernetes CLI
Along the way, you might need to install other CLIs and you can do so but just searching Homebrew install Kubernetes on google to find the install command line for that

### OpenShift Global Secret Entitlement Key
In this step, we need to add the IBM entitlement key to the OpenShift Cluster in order for your ArgoCD and Gitops service account to be able to setup the necessary databases and PV storages it needs.

- First, login to your OpenShift Cluster UI, and click on copy login command to get the login command for the cluster
- Open your terminal, and paste the login command and if you are logged in successfully, you should get a message saying you are logged into and the server details
- If you don't already have an IBM entitlement Key, then go to this URL and click add a new key
- Copy the new key, then run the following commands in order
- oc extract secret/pull-secret -n openshift-config
- export IBM_ENTITLEMENT_KEY=<add the copied key here>
```sh
oc registry login --registry="cp.icr.io" --auth-basic="cp:${IBM_ENTITLEMENT_KEY}" --to=./.dockerconfigjson

oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./.dockerconfigjson
```
- Make sure that you get this message "
secret/pull-secret data updated" and that means your commands have successfully updated the IBM entitlement key 
- If possible, we need to reload all working nodes on the cluster and we do that by logging into the ROKS cluster and selecting all working nodes and clicking reload (this step will take about 5 - 15 minute and you have to wait until all nodes are back up and the status shows normal again)


## Deploying the B2BI Sterling File Gateway in a single namespace (i.e. `b2bi-prod`): 

Infrastructure
### Infrastructure - Kustomization.yaml

-	Edit the Infrastructure layer `${GITOPS_PROFILE}/1-infra/kustomization.yaml`
-	un-comment the following lines, commit and push the changes and synchronize the `infra` Application in the ArgoCD console

    ```bash        
    cd multi-tenancy-gitops/0-bootstrap/single-cluster/1-infra
    ```
    - Kustomization.yaml

    - argocd/consolenotification.yaml
    - argocd/namespace-b2bi-prod.yaml
    - argocd/namespace-sealed-secrets.yaml
    - argocd/serviceaccounts-b2bi-prod.yaml
    - argocd/sfg-b2bi-clusterwide.yaml
    - argocd/daemonset-sync-global-pullsecret.yaml
    

### Services - Kustomization.yaml

This recipe is can be implemented using a combination of storage classes. Not all combination will work, the following table lists the storage classes that we have tested to work:

    | Component | Access Mode | IBM Cloud | OCS/ODF |
    | --- | --- | --- | --- |
    | DB2 | RWO | ibmc-block-gold | ocs-storagecluster-cephfs |
    | MQ | RWO | ibmc-block-gold | ocs-storagecluster-cephfs |
    | SFG | RWX | managed-nfs-storage | ocs-storagecluster-cephfs |

-	Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` and install Sealed Secrets by uncommenting the following line
-	**commit** and **push** the changes and refresh the `services` Application in the ArgoCD console.

    Kustomization.yaml 
    - argocd/instances/sealed-secrets.yaml
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` & sync ArgoCD. 

1. Clone the services repo for GitOps, open a terminal window and clone the `multi-tenancy-gitops-services` repository under your Git Organization.
        
    ```bash
    git clone git@github.com:${GIT_ORG}/multi-tenancy-gitops-services.git
    ```

Modify the B2BI pre-requisites components which includes the secrets and PVCs required for the B2BI helm chart.

-	Go to the `ibm-sfg-b2bi-prod-setup` directory:

        ```bash
        cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi-prod-setup
        ```

-	Generate a Sealed Secret for the credentials.

        ```bash
        B2B_DB_SECRET=db2inst1 \
        JMS_PASSWORD=password JMS_KEYSTORE_PASSWORD=password JMS_TRUSTSTORE_PASSWORD=password \
        B2B_SYSTEM_PASSPHRASE_SECRET=password \
        ./sfg-b2bi-secrets.sh
        ```

Note: Generate Persistent Volume Yamls required by Sterling File Gateway (the default is set in RWX_STORAGECLASS environment variable to `managed-nfs-storage` -if you are installing on ODF, set `RWX_STORAGECLASS=ocs-storagecluster-cephfs`)

        ```bash
        ./sfg-b2bi-pvc-mods.sh
        ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops-services` 

Enable DB2, MQ and prerequisites in the main `multi-tenancy-gitops` repository

Before you enable DB2, you need to change the repo as shown in the screenshot below:
- argocd/instances/ibm-sfg-db2-prod.yaml

 

-	Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following lines to install the pre-requisites for Sterling File Gateway

        kustomization.yaml 
        - argocd/instances/ibm-sfg-db2-prod.yaml
        - argocd/instances/ibm-sfg-mq-prod.yaml
        - argocd/instances/ibm-sfg-b2bi-prod-setup.yaml
        ```

  **Optional** Modify the DB2 and MQ storage classes to the environment that you use, the files are in `${GITOPS_PROFILE}/2-services/argocd/instances`. Edit `ibm-sfg-db2-prod.yaml` and `ibm-sfg-mq-prod.yaml` to switch the storageClassName if necessary.


    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` and
    > sync the ArgoCD application `services`.
    
    > Make sure that the sterling toolkit pod does not throw any error.
    > Wait for 5 minutes until the database is fully initialized. 
   

Generate Helm Chart values.yaml for the Sterling File Gateway Helm Chart in the `multi-tenancy-gitops-services` repo; 

Note: the default storage class is using `managed-nfs-storage` - if you are installing on ODF, set `RWX_STORAGECLASS=ocs-storagecluster-cephfs`.

    ```
    cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi-prod
    ./ibm-sfg-b2bi-overrides-values.sh
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops-services` 

-	Edit the Services layer `${GITOPS_PROFILE}/2-services/kustomization.yaml` by uncommenting the following line to install Sterling File Gateway
-	**commit** and **push** the changes and refresh the `services` Application in the ArgoCD console:

    kustomization..yaml
    - argocd/instances/ibm-sfg-b2bi-prod.yaml
    ```

    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` and
    > sync ArgoCD application `services` this will take around 1.5 hr for the database setup.

---

> **⚠️** Warning ** Important:  
> If you decided to scale the pods or upgrade the verison you should do the following steps:
>> **This is to avoid going through the database setup job again**
>> YOU SHOULD DO THIS STEP BEFORE YOU DO ANY CHANGES OR MODIFIACTIONS TO YOUR APPLICATIONS

- Step 1:
    ```bash
    cd multi-tenancy-gitops-services/instances/ibm-sfg-b2bi-prod
    ```
- Step 2:
  - Inside `values.yaml`, find & set 
        ```bash
        . . .
        datasetup:
          enabled: false
        . . .
        dbCreateSchema: false
        . . .
        ```
- Commit and push the changes for the `multi-tenancy-gitops-services` repo.
---

Validation

-	Retrieve the Sterling File Gateway console URL.

    ```bash
    oc get route -n b2bi-prod ibm-sfg-b2bi-prod-sfg-asi-internal-route-dashboard -o template --template='https://{{.spec.host}}'
    ```

-	Log in with the default credentials:  username:`fg_sysadmin` password: `password` 


Additional instance of Sterling File Gateway

The current setup has an additional set of customized instance of Sterling File Gateway B2BI in `b2bi-nonprod` namespace. Follow the similar proceure above to run the updates for the `b2bi-nonprod` namespace.


## LightWell Deployment

- Copy 2 files from the resources in multi-tenecy-gitops to your new muti-tenecy-gitops repo under bootstrap/single-cluster/2-services/argocd/instances
```
  - Ibm-lw-db2.yaml
  - Lightwell-framework.yaml
  ```
-	Add deployment code to the Kustomization.yaml under bootstrap/single-cluster/2-services/
```
  - # lightwell
  - # - argocd/instances/ibm-lw-db2.yaml
  - # - argocd/instances/lightwell-framework.yaml
  ```
- Copy the 2 folders below to the muti-tenecy-gitops-services/instances
```
  a.	ibm-lw-db2
  b.	lightwell-framework-instance
  ```
4-	uncomment the LW db line in your Kustomization.yaml
```
  - argocd/instances/ibm-lw-db2.yaml
```
    >  💡 **NOTE**  
    > Commit and Push the changes for `multi-tenancy-gitops` & sync ArgoCD. 


- Under the services repo, open the lightwell-framework-instance
- Under configmap, open the customer_LW_license_properties and modify the customer name and license
- Open the application.properties and modify,
a.	All URLs to match your cluster
b.	All usernames and password if not using the defaults listed
c.	B2bi and LW databases IP addresses and port number

-	Then follow the LW installation guide for the rest of the deployment (please skips steps you already done previously)

- Generate required k8s Secret
- Update application.properties with the B2Bi and Database connection properties
- Update customer_LW_license.properties with the license key
- login to the cluster before you run the java commands
- Generate "encryption.key" and "portal.key" using the LwEncryption.jar

```bash
- java -jar LwEncryption.jar -k portal.key
- java -jar LwEncryption.jar -k encryption.key
```

- Update YAMLs
- Update YAMLs
- Update ${STORAGECLASS} in 3-Installation/lightwell-framework-instance/pvcs/lwfw-files-pvc.yaml
- Update "image" as required in 3-Installation/lightwell-framework-instance/statefulsets/lwfw-statefulset.yaml
```bash
- Generate a SealedSecret (requires kubeseal cli)
- ./3-Installation/Lightwell-Framework-Secrets/lwfw-secret.sh (./lwfw-secret.sh)
```
- Copy the generated SealedSecret into the Kustomize structure
cp 3-Installation/Lightwell-Framework-Secrets/lw-app-prop.yaml 3-Installation/lightwell-framework-instance/secrets/lw-app-prop.yaml
- Validate YAMLs
```bash
oc apply -k 3-Installation/lightwell-framework-instance --dry-run=client -n <namespace>
```
- Version control yamls in SCM
- Deploy YAMLs
- Deploy YAMLs
```bash
oc apply -k 3-Installation/lightwell-framework-instance -n <namespace>
```

- (Optional) Deploy using ArgoCD.  A sample ArgoCD application (3-Installation/lightwell-framework.yaml) has been provided.

### Post installation Config 

- Edit the `admin` account and grant it with the "APIUser" permission to be able to log in to B2Bi Customization > Customization view.
- Accounts > User Accounts and search for "admin".
- Grant it the "APIUser" permission in the "Permission" section, click save and Finish
- Log in to Customization > Customization view
- Click on CustomJar tab and click "Create CustomJar" button
  -	Vendor Name: LW
  - Vendor Version: 1.0
  -	Jar Type: Library
  - File: LWUtility.jar
  - Target Path: DCL
- Click on "Save CustomJar" button
- Click on CustomService tab and click "Create CustomService" button
  - Service Name: LW-RuleService
  - File: LwRuleService.jar
- Click on "Save CustomService" button
- Click on CustomService tab and click "Create CustomService" button
  - Service Name: LW-UtilityService
  - File: LwUtilityService.jar
- Click on "Save CustomService" button
- Click on CustomService tab and click "Create CustomService" button
- Service Name: LW-UtilsExternal
- File: LwUtilsExternal.jar
- Click on "Save CustomService" button
- Click on PropertyFile tab and click "Create PropertyFile" button
  - PropertyFile Prefix: customer_LW
  - Property File: customer_LW.properties
- Click on "Save PropertyFile" button
- Update customer_overrides.properties for LW deployment (ie. set DB configurations)
- Click on PropertyFile tab and edit "customer_overrides" PropertyFile
- Click on "customer_overrides" and select "General" tab
- Click on "Edit" button
  - Property File: customer_overrides.properties
-	Click on "Replace Existing Property File" checkbox
-	Click on "Save PropertyFile" button
-	Edit "customer_LW" Property File
-	Click on "customer_LW" Property File and select "Property" tab
-	Modify the following properties:
    -	Property: DefaultEmail
    -	Property Value: <Email>
    -	Property: DefaultEmailSender
    -	Property Value: <Email>
    -	Property: ErrorAckEmail
    -	Property Value: <Email>
    -	Property: OverdueAckEmail
    -	Property Value: <Email>
    -	Property: PortalAuditEmail
    -	Property Value:: <Email>
    -	Property: DatabaseStorage
    -	Property Value: true
    -	Property: TempDirectory
    -	Property Value: /files
    -	Property: ArchiveOlderThanDays
    -	Property Value: <Days before archive>
    -	Property: ArchiveRootDirectory
    -	Property Value: /files/archive
    -	Log in to B2Bi Console
    -	Deployment > Resource Manager > Import/Export
    -	Select "Import"
    -	File Name: EnvelopeExport.xml
    -	Passphrase: password
    -	Import All Resources: Select checkbox
    -	Click Next x3, Finish
    -	Select "Import"
    -	File Name: FrameworkExport.xml
    -	Passphrase: password
-	Import All Resources: Select checkbox
-	Click Next x3, Finish
-	Issue: File too big so had to restart the application for customer_overrides to take effect
-	From B2Bi console, Operations > System > Troubleshooter > Stop the System
-	From the `asi` pod, go to the terminal window
```bash
cd ibm/b2bi/install/bin
./hardstop.sh
./run.sh
```
>	Note: This will install the CustomJar and CustomServices along with the customer_overrides
www.	Issue: Stopping the B2Bi service will cause the STS probes to restart the pod as its failing the check
-	Increase the configuration of the probes or remove as a workaround
>	NOTE: FrameworkExportSPE.xml is only used if ITXA is installed
-	Log in to B2Bi Console
-	Deployment > Services > Configuration
-	Search SMTP
-	Edit LW_SA_SMTP as needed
-	Click Save
- Search Portal
-	Edit LW_SA_HTTP_S1N1LOCAL_PORTAL
-	HTTP Listen Port: 5580
-	Click Save
-	Search b2bi
-	Edit LW_SA_LWJDBC_B2BI
-	Pool Name: db2Pool
-	Click Save

### validation
- Validate Lightwell Framework deployment
- Log in to Lightwell Portal and check Framework Version
-	Framework Management > Rules > Route Rules
-	Click New and set the following:
-	Document Type: 850
-	Click Save
-	Framework Management > Rules > Send Rules
-	Click New and set the following:
-	Document Type: 850
-	Send BP: LW_BP_SEND_EMAIL
-	Subject Mask: 850
-	Email Recipient: <Email>
-	Click Save
-	Framework Management > Test Flow
-	Protocol: PROXY
-	User ID: admin
-	File: Partner1_8850.edi
-	Click Submit
-	Document Visibility > Submit File to B2Bi
    -	Protocol: PROXY
    - User ID: admin
    - File: Partner1_850.edi
- Click Submit
- Click "Show Documents"
