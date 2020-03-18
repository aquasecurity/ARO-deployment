# Implement Aqua on ARO (Azure Red Hat OpenShift)

> The following instructions are compatible with Aqua Cloud-Native Security Platform 4.2+ and ARO 3.11

## Introduction
Follow the procedure on this page to perform a standard deployment of Aqua CSP in an OpenShift cluster. The Aqua Server components are deployed as Pods and Services, while the Aqua Enforcer is deployed as a DaemonSet.

This procedure describes how to deploy Aqua components on OpenShift through the OpenShift command line (oc utility). It assumes that the Aqua images will be used directly from the private Aqua Security registry on Docker Hub.

## Step 1: Prepare prerequisites
Perform the following prerequisite steps before you deploy Aqua components.

1. Log in to the cluster as a user with *ARO Customer Admin* privileges

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

4. Set the Aqua Enforcer priviliges
```
oc adm policy add-cluster-role-to-user customer-admin-cluster system:serviceaccount:aqua-security:aqua-account
oc adm policy add-scc-to-user privileged system:serviceaccount:aqua-security:aqua-account
oc adm policy add-scc-to-user hostaccess system:serviceaccount:aqua-security:aqua-account
```

5. Create a secret to store the credential to pull Aqua's images from the registry. Replace the key holders below with the credential you received from Aqua Security
```
oc create secret docker-registry aqua-registry --docker-server=registry.aquasec.com --docker-username=<AQUA_USERNAME> --docker-password=<AQUA_PASSWORD> --docker-email=no@email.com -n aqua-security
```

6. Create a secret to store the database password. Replace the key holders below with your choice for the database password
```
oc create secret generic aqua-database-password --from-literal=db-password=<DB_PASSWORD> -n aqua-security 
```

## Step 2: Deploy the Aqua Server, Database, and Gateway

1. Download the **aqua-db.yaml**, **aqua-console.yaml**, **aqua-gateway.yaml** files

2. In case there are DNS resolution issues, you might need to replace all instances of aqua-db with the IP address of the Aqua Gateway service.

3. Deploy all components
```
oc project aqua-security
oc create -f aqua-db.yaml
oc create -f aqua-console.yaml
oc create -f aqua-gateway.yaml
```

4. Run **oc status** to verify the deployment of all components, and to capture the IP address assigned to the Aqua Gateway. You will need it when deploying the Aqua Enforcer. Your console output should show a line like the following, which includes the IP address of the Aqua Gateway:
```
svc/aqua-gateway - 172.30.100.187:3622
```
To get Aqua's external URL address run
```
oc describe route aqua-web -n aqua-security
```

## Step 3: Activate Aqua
1. Open your browser and navigate to the external URL generated in the route. Remember to use HTTPS.
```
https://aqua-web.apps....
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
3. On the **Enforcers > Create new group** screen, fill in the setting and make surethe **Orchestrator** is set to **OpenShift** (for more information about setting up Enforcers, please read Aqua's documentation)
4. Click **Create Group** and wait for the server acknowledgment
5. Copy the DemonSet YAML text from the screen (choose the 'Copy to clipboard' option in the UI)
6. Save the copied YAML text to a file aqua-enforcer.yaml on your host
7. Check the file and make sure that it uses the right namespace, account name, image names, registry, and token infomrtaion
8. Create the deployment by running the following commnads:
```
oc project aqua-security
oc create -f aqua-enforcer.yaml
```
9. It could take several minutes for the Aqua Enforcer(s) to be installed. Use the following command to  monitor the status of the Aqua Enforcer deployment -
```
oc get pods -n aqua-security
```

## Step 5: Verify Aqua's Enforcer
1. In the Aqua UI: Navigate to Enforcers and expand the line with the name of the new Enforcer. If the Enforcer is colored green then your installation is ready.
