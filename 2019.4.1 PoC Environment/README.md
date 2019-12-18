# Cloud Pak for Integration 2019.4.1 Installation on AWS

Version 2019.4.1 of IBM CP4I requires Openshift 4.2 which is a significant change from previous versions.  The OCP installation can auto-provision a cluster on a number of different providers, including AWS which will be the focus of this document.

The cluster created by these instructions has one OCP master, one ICP/Cloud foundation node, and three worker/compute nodes.  This allows resilience to be demonstrated by shutting down a worker node.

## Configure your AWS account

For OCP 4.2 you will need your own domain name for the cluster.  This can be an existing domain name or a newly created one.  For simplicity these instructions will create a new one.

1. Sign into the AWS console then under 'Services' select Route 53 from 'Networking and content delivery'
2. Type a name into the box and click 'Check'.
3. Click through the process to register your domain name.
4. When this is done, go back to the Route 53 dashboard and select 'Hosted zones' on the right.
5. Click on 'Create Hosted Zone'.  On the right, type in your domain name in the box and leave the type on 'Public Hosted Zone'.  Then click 'Create'

For security, you need to create an administrative user for the purposes of creating a cluster, rather than using your own root account.

6. In the AWS console under 'Services' select 'IAM' from 'Security, Identity and Compliance'
7. On the left hand side choose 'Users' then 'Add User' at the top.
8. Type in a username, then select 'Programmatic access'.
9. Select 'Attach existing policies directly'.
10. Check 'AdministratorAccess' in the list and click 'Next'.
11. You do not need to add tags.  Click 'Next' and then 'Create user'
12. Now make a note of the Access key ID and the Secret access key.

## Set up your Red Hat account

You will need a Red Hat customer account.  Visit cloud.redhat.com/openshift to access the cluster management page and log in with your credentials.  If you do not have access, contact Red Hat.

1. Click on 'Create Cluster'
2. Select 'Red Hat OpenShift Container Platform'.
3. Choose AWS for your provider.
4. Choose 'Installer-Provisioned Infrastructure'.

The next page displays the things you need to run the installer.  The binaries will need to be run on the installer node you will create later so you will need to use scp to copy them across.

5. Download]the installer for Linux: 	openshift-install-linux-4.2.10.tar.gz
6. Download and save the pull secret.
7. Download the CLI tools for Linux: openshift-client-linux-4.2.10.tar.gz

## Install OCP

1. Provision an installer node in AWS.  This should ideally have 8 cores, so select t2.2xlarge for the instance type.  It will work even with a t2.micro single core instance but it will take a very long time.  It should have 150Gb of EBS storage.  For the OS image choose Amazon Linux.  Other distributions will work, but these instructions refer to Amazon Linux.  RHEL 8 is different because the default container tool (installed with yum install docker) is actually podman.  There is currently a bug in podman that prevents the CP4I installer working. RHEL 8 can be used if true docker is installed.
2. Copy the openshift installer to the installer node, extract the single binary file and make it executable.
3. Copy the CLI tools archive and extract the two binaries: oc and kubectl.  Make them executable then copy them to /usr/bin (or some other location if you wish to amend your PATH variable)
4. Install docker: `yum install docker`


5. Run `openshift-install create install-config --dir=<install directory>` where `<install directory>` is the name for a directory that will be created to store the generated installation configuration file called install-config.yaml
- Select AWS for the platform.
- Select a region, preferably reasonably local since it reduces latency.
- For your base domain select the domain name you created earlier.  It should be one of the pre-populated options.
- Enter a name for your cluster, anything will do
- When prompted for the pull secret, paste in the secret from the earlier AWS configuration step.

6. We will be configuring one master node and four compute nodes - one of which will host the IBM Cloud Foundation (ICP) components.  Edit install-config.yaml in your install directory as follows:
7. Find the 'controlPlane' section of YAML.  Change the number of replicas to 1 and add the AWS instance type of t2.2xlarge as follows:

```controlPlane:
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      type: t2.2xlarge
  replicas: 1
  ```

8. Find the 'compute' section of the YAML.  Change the number of replicas to 4 and add the AWS instance type of t2.2xlarge, as follows:

```compute:
- hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: t2.2xlarge
  replicas: 4
  ```

9. Run the installer with the command `openshift-instal create cluster --dir=<install directory>`
10. The installer will complete with a message displaying temporary credentials and URLs to log into the cluster.  Save this bit of text for later.
11. When you log in with kubeadmin you will get a message about kuebadmin being a temporary user and to set up an oauth provider.  If you want a quick and easy oauth provider, click the link to set one up, and select htpasswd.  You will be prompted for an htpasswd file, generate one as follows
- Install htpasswd with `yum install httpd-tools`
- Run `htpasswd -c htpasswd.txt admin` and then supply a password.  The resulting htpasswd.txt can be uploaded to the OCP GUI as above.  Make a note of the password, you will need it during CP4I installation.
- Give your new admin user the cluster-admin role as follows.  Set up the CLI by running the export KUBECONFIG command that you saved in step step 13.
- Run `oc login` and log in with the kubeadmin user and temporary password.
- Run `oc adm policy add-cluster-role-to-user cluster-admin admin`.  Then run `oc login` with the admin user you created, and you should have cluster admin privileges.  Verify this by typing `oc projects` and seeing lots of projects listed.

## Install CP4I

1. Download the offline installer from Fix Central and unzip.
2. Before the installation you will need to upload the installation images to the Openshift internal registry. For this to work you need to expose the default registry route.  Run `oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge`.
3. Docker does not allow you to override the self-signed certificates, unlike podman which comes with RHEL 8.  To get around this you will need to install the root CA, as detailed below.
4. Obtain the route to the registry: `oc get routes -n openshift-image-registry` and make a note of the host.
5. List the certificate chain with `openssl s_client -connect default-route-openshift-image-registry.apps.test-cp4i4.bencornwell.co.uk:443  -showcerts`
6. The root CA certificate has a common name (CN) something like ingress-operator@12345678, and is probably the second certificate in the displayed output.  Copy the certificate which is the bit between and including the BEGIN CERTIFICATE and END CERTIFICATE lines, and save to a file <cert file>.
7. Copy the certificate file to the right place: `cp <cert file> /etc/pki/ca-trust/source/anchors/`
8. Update the trust store: `update-ca-trust extract`
9. You can then log into docker with your admin creds: `oc login`
10. Log into docker, obtaining the credentials and host from the cluster: `docker login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')`
11. Extract the CP4I installation file: `tar xf ibm-cloud-pak-for-integration-x86_64-2019.4.1.1-offline.tar.gz`
12. Copy the kubeconfig file from your install directory from the OCP installation to the CP4I installation directory: `cp <install dir>/auth/kubeconfig installer_files/cluster/kubeconfig`.  Do not use `oc config view` as suggested in the CP4I knowledge centre.
13. Change to the cluster directory `cd installer_file/cluster`
14. List your nodes with `oc get nodes`
15. Choose one of your compute nodes for master, proxy and management.  Make a note of its hostname (it will be an internal hostname starting with ip-).
16. Edit the config.yaml file
17. In the cluster_nodes section put only the hostname you chose above under master, proxy and management, e.g.

```cluster_nodes:
  master:
    - ip-10-1-2-3.eu-west-2.compute.internal
  proxy:
    - ip-10-1-2-3.eu-west-2.compute.internal
  management:
    - ip-10-1-2-3.eu-west-2.compute.internal
```

18. Add your admin password for the OCP cluster in the default_admin_password line as follows:
```
default_admin_password: mypass
   - '(.*)'
```

  Make sure that the password is on the same line as default_admin_password exactly as shown.

19. Start the installer with `docker run -t --net=host -e LICENSE=accept -v $(pwd):/installer/cluster:z -v /var/run:/var/run:z -v /etc/docker:/etc/docker:z --security-opt label:disable ibmcom/icp-inception-amd64:3.2.2 addon`

    It should take approx 1.5-2 hours if you have chosen an 8 core node for the installer.

Troubleshooting tips are in troubleshooting.md

To deploy capabilities, see <tbc>
