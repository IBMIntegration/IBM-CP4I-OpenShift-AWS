# Cloud Pak for Integration 2019.4.1 Installation on AWS

Version 2019.4.1 of IBM CP4I requires Openshift 4.2 which is a significant change from previous versions.  The OCP installation can auto-provision a cluster on a number of different providers, including AWS which will be the focus of this document.

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

1. Provision an installer node in AWS.  This should ideally have 8 cores, so select something like t2.large instance.  It will work even with a t2.micro single core instance but it will take a very long time.  It should have 150Gb of EBS storage.  For the OS image choose Amazon Linux.  Other distributions will work, but these instructions refer to Amazon Linux.  RHEL 8 is different because the default container tool (installed with yum install docker) is actually podman.  There is currently a bug in podman that prevents the CP4I installer working. RHEL 8 can be used if true docker is installed.
2. Copy the openshift installer to the installer node, extract the single binary file and make it executable.
3. Copy the CLI tools archive and extract the two binaries: oc and kubectl.  Make them executable then copy them to /usr/bin (or some other location if you wish to amend your PATH variable)
2. Install docker: `yum install docker`
3. Log onto the cluster manager: https://cloud.redhat.com/openshift/
4. Select 'Create cluster' then 'Create a cluster using the command line'
5. Select AWS then 'Installer provisioned infrastructure'
7. During these instructions you create an AWS IAM user and an access key which has credentials.  You will need these.
7. Once your RedHat account is configured with your AWS account, on the cluster management page should contain a link to download the installer.  Put this file on your installer node.
8. From the same page download your pull secret and save it somewhere.
9. Finally download the command line tools (oc) and copy them to /usr/bin on the installer node.
10. Following the instructions on https://docs.openshift.com/container-platform/4.2/installing/installing_aws/installing-aws-customizations.html create the installation config file `install-config.yaml`
11. Customise the install-config.yaml file following the instructions.  You will need to set up four t2.2xlarge instances for the compute nodes, and one t2.2xlarge for the master node.  One of the compute nodes will become the ICP master, management and proxy nodes.
12. Run the installer.
13. The installer will complete with a message displaying temporary credentials and URLs to log into the cluster.  Save this bit of text for later.
14. When you log in with kubeadmin you will get a message about kuebadmin being a temporary user and to set up an oauth provider.  If you want a quick and easy oauth provider, click the link to set one up, and select htpasswd.  You will be prompted for an htpasswd file, generate one as follows:
15. Install htpasswd with `yum install httpd-tools`
16. Run `htpasswd -c htpasswd.txt admin` and then supply a password.  The resulting htpasswd.txt can be uploaded to the OCP GUI as above.  Make a note of the password, you will need it during CP4I installation.
17. Give your new admin user the cluster-admin role as follows.  Set up the CLI by running the export KUBECONFIG command that you saved in step step 13.
18. Run `oc login` and log in with the kubeadmin user and temporary password.
19. Run `oc adm policy add-cluster-role-to-user cluster-admin admin`.  Then run `oc login` with the admin user you created, and you should have cluster admin privileges.  Verify this by typing `oc projects` and seeing lots of projects listed.

## Install CP4I

1. Download the offline installer from Fix Central and unzip.
2. Before the installation you will need to upload the installation images to the Openshift internal registry. For this to work you need to expose the default registry route.  Run `oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge`.

NOTE STEPS 3-8 THIS MAY NOT BE NECESSARY FIRST TIME AROUND

3. Docker does not allow you to override the self-signed certificates.  Install the root CA as follows:
4. Obtain the route to the registry: `oc get routes -n openshift-image-registry` and make a note of the host.
5. List the certificate chain with `openssl s_client -connect default-route-openshift-image-registry.apps.test-cp4i4.bencornwell.co.uk:443  -showcerts`
6. The root CA has a common name (CN) something like ingress-operator@12345678.  Copy the certificate that follows, between AND INCLUDING the -----BEGIN CERTIFICATE----- lines and save to a file.
7. `cp <cert file> /etc/pki/ca-trust/source/anchors/`
8. `update-ca-trust extract`
9. You can then log into docker with your admin creds: `oc login`
10. `HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')`
11. `docker login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $HOST`
12. Now follow the instructions in https://www.ibm.com/support/knowledgecenter/en/SSGT7J_19.3/install/install_red_hat.html bearing in mind the following:
13. Copy the kubeconfig file from your installation directory from the OCP installation.  Do not use `oc config view`
14. Choose one of your compute nodes for master, proxy and management
15. You don't need to label any nodes.
16. Your router domain should be apps.mydomain.com so the cluster_host and proxy_host are icp-console.apps.mydomain.com and icp-proxy.apps.mydomain.com respectively. ??
17. Add your admin password for the OCP cluster in the default_admin_password line as follows: `default_admin_password: <password>` and then under password_rules only have one line: `   - '(.*)'`
18. Start the installer with `docker run -t --net=host -e LICENSE=accept -v $(pwd):/installer/cluster:z -v /var/run:/var/run:z -v /etc/docker:/etc/docker:z --security-opt label:disable ibmcom/icp-inception-amd64:3.2.2 addon`

Troubleshooting tips are in troubleshooting.md
