# oc-mirror

= Installing oc-mirror on a connected node 

== Prerequisites: 
- Docker/Podman
- Go  (sudo dnf install go)
- Make (sudo dnf install make)
----
Note: You must have a config.json in your .docker/ directory in order to successfuly pull and push the mirror images ( this is applicable for all nodes), it is important because oc-mirror uses docker containers to pull images from a known registry and then uses a separate set of containers to push to a local or remote repository of your choosing. 
----

== Cloning the repository

* The repository is located at https://github.com/openshift/oc-mirror :
     - $ git clone https://github.com/openshift/oc-mirror.git 

* Please verify in your home directory that you have a .docker/config.json file that allows you to push and pull container images. (just copy and paste your keys in docker hub.) 

== Building the oc-mirror operator
* In the oc-mirror directory that was cloned, you will run the following command:
----
    $ make build
----

* The operator should be able to run the imageset-config.yml from the oc-mirror directory unless specified in an absolute path.


== Using the operator 
* In this use case, we will be pulling operator images for OCP disconnected.
    - The oc-mirror reads yaml files to gather information such as index(s) i.e redhat-certified and  community, the operators you want to pull and specific versions of those operators
* The first step is to edit the imageset-config.yml to reference what operators you wish to mirror. 

Example below:
----
apiVersion: mirror.openshift.io/v1alpha1
kind: ImageSetConfiguration
mirror:
  ocp: 
  operators: 
    - catalog: registry.redhat.io/redhat/redhat-operator-index:4.9
      headsonly: false
      packages: 
      - name: rhacs-operator
         startingVersion: '3.67.2'
----

* Running oc-mirror
    - Once you are in the oc-mirror directory you will uyse the following command to gather the operators: 
----
     $ ./bin/oc-mirror --config imageset-config.yaml file://archive
 
     Note: This will set of the oc-miorror command to configure via the imageset-config that we configured above. The file://archive is the destination that oc-mirror sends the tar files and workspaces to. Dont worry about the archive directory being on the system, oc-mirror will create it for you. 
----

* Completed Mirror
     - When oc-mirror is completed you will then navigate to the archive directory specified above. 
     - To verify that it has worked, you will see your newly created archive directory. Inside that directory, you will find a mirror_seql_000000.tar file. 

* Deployment prep
     - Now that you have successfully used oc-mirror to package the images, you are ready to deploy them to your registry. 
     - To do so you must package the oc-mirror directory into a tar file.
----
    $ tar -cvf oc-mirror.tar oc-mirror/
----

* Transfer the tar file to the destination network

----
Note: the recommended way to transfer this is via aws for speed. 

    $ scp oc-mirror.tar user@destination.node:/userdir/

OR (if applicable)
    
   1.) [localhost]$ aws s3 cp oc-mirror.tar s3://tempXXX/oc-mirror.tar

   2.) [destinationhost] $ aws s3 cp s3://tempXX/oc-mirror.tar .

 


== Deploying to the registry

* Unpack the oc-mirror.tar

---- 
    $ tar -xvf oc-mirror.tar
----

* Verify the tool is operable

----
    $ ./bin/oc-mirror --help
----

* Verify that your config.json is in ~/.docker/ and has push,pull permissions

* Load image to registry

----
    $ ./bin/oc-mirror --from archives/mirror_seql_000000.tar docker://my.custom.registry.com:5000

Note: 
    this will push to your registry and at the end of the output, oc-mirror will give you paths to your index.yaml(s). typically they will be located in oc-mirror-workspace/results-XXXXXX/

The output will have a big blue "INFO" at the end to easily locate the path. 
----

== Additional notes

* to push to a remote registry, all server certificates must be verified by a CA otherwise you must add the servers certificate to your store. 
----
$ openssl s_client --connect <yourreg> --showcerts
$ sudo trust anchor path/to/certificate.crt
    Note: if using rhel8
----

* Looking for specific operators, catalogs, or channels

----
./bin/oc-mirror list operators --catalogs --version=4.9

./bin/oc-mirror list operators --catalog=catalog-name

./bin/oc-mirror list operators --catalog=catalog-name --package=package-name 

./bin/oc-mirror list operators --catalog=catalog-name --package=package-name --channel=channel-name
----
