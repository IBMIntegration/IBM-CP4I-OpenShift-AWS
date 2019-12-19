## Troubleshooting installation of CP4I on Openshift

Be sure to give lots of CPU to your installation node otherwise it will take a very long time.  Recommend 8 cores.

If using RHEL for the install node, yum install docker actually installs podman.  The version at time of writing (10/12/2019) is bugged and won't work - it fails when trying to load the images extracted from the tar file with an error about the manifest.  You can either use RHEL 7.x or install actual docker which should work.  Podman however supports a --tls-verify=false option that lets you ignore self signed certificate issues - docker does not (see below).

Re-running the CP4I installer does not currently work, neither does uninstalling. If something goes wrong and you get stuck because of this you will need to tear the cluster down and start again (see below).

To delete the cluster and re-install:
- Run ./openshift-install destroy cluster --dir=<install directory>  AND for CP4I also delete any lock files in the install_files/cluster directory.
- Run the OCP installer again - with a clean install directory as per the instructions
- The OCP installation process apparently installs the root CA onto your OS.  If you are using genuine docker rather than podman, you cannot override the root CA check.  In that case if you re-install a new cluster and you wish to preserve your install node, install the new root CA on the installer host for the docker command otherwise you get certificate errors using docker.  Below command is for Amazon Linux or Redhat 7.x
- `openssl s_client -connect default-route-openshift-image-registry.<your apps domain>:443  -showcerts`
- copy the root CA cert from this output and save to a file
- `cp <cert file> /etc/pki/ca-trust/source/anchors/`
- `update-ca-trust extract`

The uploading of images in the CP4I installer takes so long that the ssh session may time out.  Either change the config or run the installer in the background, redirect the output to a file then run something like top in the window to keep the shell open.  You can tail the file you've redirected to to view progress.

## Installing capabilities

For the ACE dashboard, the default storage class of GP2 does not work.  Something else needs to be set up.  EFS should work.

If using offline install, change the helm repo in the advanced chart parameters to local-charts from the entitled repository, and then change the server namespace of the docker images to `image-registry.openshift-image-registry.svc:5000`.  Alternatively, simply delete the entitled repository.  You may still have to change the helm repo in the chart but the docker image namespaces should be fine.

With the offline repo (local-charts) you will need an image pull secret in 2019.4.1 to give access to the internal registry (not yet clear which secret is best to use if there is a built-in one).
