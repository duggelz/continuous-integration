# Bazel continous integration setup

This workspace contains the setup for the continuous integration
system of Bazel. This setup is based on docker images built by bazel.

Make sure you have a Bazel installed with a recent enough version of
it. Also make sure [gcloud](https://cloud.google.com/sdk/) and
[docker](https://www.docker.com) are correctly configured on your
machine.

The CI system is controlled by a Jenkins instance that is shipped in a
Docker image and run on a dedicated VM. Additional Jenkins slave might
run in that VM in dedicated Docker instances. Each slave is
controlled by the Jenkins instance and might be either Docker
instance (in which case the docker image is built by Bazel, see
`jenkins/BUILD`), GCE virtual machine (in which case the
`gce/create-vm.sh` should create it and the corresponding
`jenkins_node` should be added to `jenkins/BUILD`) or an actual machine
(in which case the corresponding `jenkins_node` should be created, the
firewall rules adapted, and the setup of the machine documented in
that README.md).

## Deploying to GCR

We do not support docker pull yet so you have to do that manually.
Fortunately a simple shell script can do that for you:

```
./jenkins/build.sh push
```

This will build your docker image for you and upload them to GCR. If
you have only modified stuff in the jenkins folder that's all that
should be needed to update the configuration.

## Setting up for local testing

 1. First, make sure the DNS "jenkins" resolves to the host machine of
    the docker installation or update the information at the top of
    the `jenkins.bzl` file.
 2. Load the docker images with `./jenkins/build.sh build`
 3. Launch the various images in your image by mapping the
    necessary folders:

```
docker run --name=jenkins-master --volume /volumes/jenkins:/var/jenkins_home \
    -p 8080:80 -p 50000:50000 -d gcr.io/bazel-public/jenkins-master
# Wait for 2-3 minutes to go until jenkins is fully started
docker run --name=deploy-slave --volume /volumes/secrets:/opt/secrets \
    gcr.io/bazel-public/deploy-slave
```

`/volumes/jenkins` will contains the project files from jenkins and
`/volumes/secrets` should be filled with the various authentication
token for the CI System: 

 - `boto_config` should be the `.boto` file generated by gcloud login
 - `github_token` should be the GitHub API authentication token
 - `github_trigger_auth_token` should contain an uniq string shared
    between GitHub and Jenkins. A GitHub webhook should be set to use
    the payload url
    `http://ci.bazel.io/job/Github-Trigger/buildWithParameters?token=TOKEN`
    where `TOKEN` is the same string as the content of the
    `github_trigger_auth_token`. This webhook should send its data in
    the `x-www-form-urlencoded` format.

If you wish to test out new configuration, you can change the security
settings in the `config.xml` file before building the docker
image. You can then access all the configuration of Jenkins. The
configuration deployed to GCR are in the various `.xml` files in the
`jenkins` package. You can set-up the ubuntu slave in the same way by
uncomenting the part for building the docker image in `jenkins/BUILD`
and `jenkins/build.sh`.

## Running the VM on GCE

We spawn these images on servers on GCE in the `bazel-public` project.
2 scripts are available to handle the VM in GCE:

 - `gce/create-vm.sh` creates the corresponding virtual machines on
   GCE.
 - `gce/delete-vm.sh` deletes the virtual machines from the GCE
   project. Be careful when doing that. It should not cause any loss
   of data as we use a permanent disk for storing Jenkins build but it
   might cause some corruption if you do it during a build. Please
   prepare Jenkins for shutdown before deleting virtual machines.

The following additional set up needs to be performed in the cloud
console to make the project work:

 1. Create a permanent disk `jenkins-volumes` for where the build of
    jenkins are constructed (secrets should be put in its `secrets`
    folder also),
 2. Create a static public IP `ci` for the jenkins front-end,
 3. The following firewall rules should be setted-up:
   - Allow all communication from the internal network (created by
     default),
   - Allow SSH (tcp 22) (created by default),
   - Allow all private network (192.168.0.0/16, 172.16.0.0/12 and
     10.0.0.0/8) to access port 50000 to `jenkins` instance (specify
     the `jenkins` tag). Also allow the public IP `ci` and any
     external slaves you might need to add,
   - Allow HTTP traffic (tcp 80) to `jenkins` tags.
