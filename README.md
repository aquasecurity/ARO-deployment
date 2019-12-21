# Implement Aqua on ARO (Azure RedHat OpenShift)

## Introduction 
Follow the procedure on this page to perform a standard deployment of Aqua CSP in an OpenShift cluster. The Aqua Server components are deployed as Pods and Services, while the Aqua Enforcer is deployed as a DaemonSet.

This procedure describes how to deploy Aqua components on OpenShift through the OpenShift command line (oc utility). It assumes that the Aqua images will be used directly from the private Aqua Security registry on Docker Hub.

## Step 1: Prepare prerequisites
Perform the following prerequisite steps before you deploy Aqua components.

1. Login in to the cluster as a user with *ARO Customer Admin* privileges

```
oc login -u <user>
```

2. Create a new project and account for the Aqua Server components: aqua-web, aqua-gateway, and aqua-db. 

```
oc new-project aqua-security
oc create serviceaccount aqua-account -n aqua-security
```

3. Annotate the SCCs you're editing. This will prevent the Sync Pod from reverting your changes.
```
oc annotate scc hostaccess openshift.io/reconcile-protect=true
oc annotate scc privileged openshift.io/reconcile-protect=true
```

4. Set the Aqua'a enforcer priviliges 
```
oc adm policy add-cluster-role-to-user customer-admin-cluster system:serviceaccount:aqua-security:aqua-account
oc adm policy add-scc-to-user privileged system:serviceaccount:aqua-security:aqua-account
oc adm policy add-scc-to-user hostaccess system:serviceaccount:aqua-security:aqua-account
```

5. Create a secret to store the credential to pull Aqua's images from the registry. Replace the key holders below with the credential you recived from Aqua Security
```
oc create secret docker-registry aqua-registry --docker-server=registry.aquasec.com --docker-username=<AQUA_USERNAME> --docker-password=<AQUA_PASSWORD> --docker-email=no@email.com -n aqua-security
```

## Step 2: Deploy the Aqua Server, Database, and Gateway

1. Download the **aqua-console.yaml** file and make the following changes -
   1. Replace <PUBLIC_IP> with the DNS name or IP address of your OpenShift master node. This address will be used to access the Aqua Server after deployment has been completed: http://<PUBLIC_IP>:30080. 
   2. Replace all occurrences of <DB_PASSWORD> with a password of your choice
   3. In case there are DNS resolution issues, you might need to replace all instances of aqua-db with the IP address of the Aqua Gateway service.

2. Download the **aqua-db.yaml** file and make the following changes -
   1. Replace all occurrences of <DB_PASSWORD> with a password of your choice
   2. In case there are DNS resolution issues, you might need to replace all occurrences of aqua-db with the IP address of Aqua Gateway service.
   
3. Download the **aqua-gateway.yaml** file and make the following changes -
   1. Replace all occurrences of <DB_PASSWORD> with a password of your choice
   2. In case there are DNS resolution issues, you might need to replace all occurrences of aqua-db with the IP address of Aqua Gateway service.
   
4. Deploy all components 
```
oc project aqua-security
oc create -f aqua-console.yaml
oc create -f aqua-db.yaml
oc create -f aqua-gateway.yaml
```
   
5. Run **oc status** to verify the deployment of all components, and to capture the IP address assigned to the Aqua Gateway. You will need it when deploying the Aqua Enforcer. Your console output should show a line like the following, which includes the IP address of the Aqua Gateway:
```
svc/aqua-gateway - 172.30.100.187:3622
```
Another option to get Aqua's console IP address is 
```
oc describe route aqua-web -n aqua-security
```

## Step 3: Activate Aqua 
1. Open your browser and navigate to the IP address of the host on which the Aqua Server is deployed.By default, port 30080 is used. If you have set up a different port, use that. Refer to System Requirements for information about port assignment.
```
http://<HOST-IP>:30080
````
When you access the Aqua Server for the first time, you must enter and confirm the password for the administrator username.
The password must be at least 8 characters long.

2. If you access Aqua for the first time, you will need to provision your License token to activate Aqua

## Step 4: Deploy Aqua enforcers
This step will deploy the Aqua Enforcer across your OpenShift cluster by using a Kubernetes DaemonSet, which automatically deploys a single Aqua Enforcer container on each node in your cluster.

First, you create a new Enforcer group in the Aqua Server. An Enforcer group is a set of zero or more Aqua Enforcers with the same configuration. You need to create one that will work with the OpenShift orchestrator; you cannot use the default Enforcer group for this.

A byproduct of the Enforcer group creation is the DaemonSet required for OpenShift. Aqua does not automatically deploy the Enforcer on the host; you do this by using an OpenShift oc create command.

You can run this command to deploy Enforcers on one or more hosts. All Enforcers deployed with the command will have the same configuration. If you need Enforcers with different characteristics, you will need to create one or more additional Enforcer groups.

1. Log in into Aqua and navigate to the **Enforcers** view
2. Click Add **Enforcer Group**
3. On the **Enforcers > Create new group** screen, fill in the setting and make surethe **Orchestrator** is set to **OpenShift** (for more information about seting up Enforcers, please read Aqua's documentation)










