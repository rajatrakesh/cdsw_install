## CDSW Installation On Cloud

This project is derived from the fantastic work done by TobyFergusion (https://github.com/tobyHFerguson). Toby wrote an extensive document on how to install CDSW early on. The document is quite technical and provides extensive coverage into how CDSW is to be installed. I aim to extend that and perhaps simply certain steps, highlight a few gotchas that I encountered in the process. 

The aim of this following document is to simply the creation and deployment of the CDSW cluster (including kerberos) on Cloud Platforms (AWS, Azure and GCP) using Cloudera Director. 

The approach for deployment remains largely the same so with minor changes, you would have flexibility to change and deploy CDSW on any of the Cloud platforms. **Note for this document, I've current done extensive testing with AWS. I will update the Azure and GCP segment and configuration in the coming weeks as well.  

The total duration of the build would be around an hour end-to-end and I have made some edits in the scripts to automate certain steps which were previously manual. 

## Pre-Requsities

Needless to say, before you start you must have access to respective cloud platform with adequate credits to deploy CDSW/CDH Cluster. At the very minimum, we will deploy (3+1+1+3) nodes of various shapes, so this may need adequate credits based on the shape that you use. 

Our basic workflow would be as follows:

* Deploy a single EC2 instance and configure Diretor 2.8 (This is the current tested release. Going forward, I will update this document to include Altus Director v6.1)
* Download this github repo on to your local system
* Update the required configuration settings, property files. that reflect your enviroment
* Add the necessary SECRET and ssh key files
* Upload all the necessary configuration files needed for Director to create a deployment (All the files in this github repo)
* Execute the Bootstrap from Director
* Test and Validate

**Other Tools**
* Putty/Terminal for ssh
* Textedit or Textwrangler for editing property files

## Instance Shapes / Cost

For the deployment, I would be using the following shapes in AWS:

* Director 	( 4CPU, 16 GB RAM) 		- m4.xlarge (xx/hr)
* CDH Master 	( 4CPU, 16 GB RAM) 		- m4.xlarge (xx/hr)
* CDH Workers	( 4CPU, 16 GB RAM)		- m4.xlarge (xx/hr)
* CDSW Master	(16CPU, 64 GB RAM)		- m4.xlarge (xx/hr)
* CDSW Worker	(16CPU, 64 GB RAM)		- m4.xlarge (xx/hr)

For all instances, I will be using the following AMI - ```ami-78485818```
All instances will be launched in the region : ```us-west-1``` which is also the cheapest option based on instance cost.

## Step 1: Director Step

For this segment, to demonstrate, I will leverage AWS to deploy this cluster. Let's provision an instance, and deploy director. For director setup, you can simply execute the script install_director.sh. This does not require any changes/edits and will deploy Director. I've added a few screenshots below to highlight some of the crucial steps. 

![Launch instance](./images/director-01.jpg)

Goto the EC2 screen and launch an instance in the ```us-west-1``` region for the ```ami-78485818```. 

![Select instance shape](./images/director-02.jpg)

At the very minimum, select the ```m4.xlarge``` shape (4CPU, 16 GB RAM) instance. 

![Select instance shape](./images/director-03.jpg)

In this screen, select the VPC that you may have created before to launch this instance. If you don't have a VPC yet, then you can create one at this stage. 

![Disk space](./images/director-04.jpg) 

Increase the size of the disk from 8 GB to 40 GB and select the 'Delete on Termination' Flag. This will ensure that any volumes attached to this instances are also deleted when this instance is terminated. 

![Assign Tags](./images/director-05.jpg)

Configure any tags that you may want to assign to your instance. These are quite handy esp. if you are using a shared environment where multiple instances are being run by other users and if you want to filter the ones that you have quickly. 

![Security Groups](./images/director-06.jpg)

In this screen, you need to setup the security groups and open the necessary ports. AWS instances would not allow inbound access to to the specific ports required if access is not explictly enabled. 

The ports that need to be opened are as follows:

* 22 - SSH
* 80 - CDSW
* 7180 - Cloudera Manager
* 7189 - Cloudera Director

![Open ports](./images/director-07.jpg)
![Open ports](./images/director-08.jpg)
 
Finally, review and launch your instance. Note the warning that credits (paid) would be consumed in your account for the duration this instance runs. 

![Open ports](./images/director-09.jpg)

The last step is to select a key pair (if one exists in your account). You can also create a key pair and assign it to your instance in this screen. Please do remember to download the key paid once you have created it. This is the time when you would be able to download the key pair after creating it. This private key would be required for you to ssh into your vm instance. 

## Configuration Files & Project Structure

There are a number of configuration files that are required and needed for this setup:

		/
		-->common.conf
		-->aws.conf
		-->install_director.sh
		-->install_mit_kdc.sh
		-->install_mit_client.sh
		/aws
		-->aws.conf
		-->cluster.configs.HDFS.core_site_safety_valve.conf
		-->instances.conf
		-->kerberos.properties
		-->provider.conf
		-->provider.properties
		-->ssh.properties
		-->SECRET.properties
		-->**public key**
		-->**private key**
		/scripts
		-->cdsw_postcreate.sh
		-->cdswmaster-bootstrap-script.sh
		-->clean-yum-cache.sh
		-->java8-bootstrap-script.sh
		-->jdk_security.sh
		-->kerberos_client.sh
		-->ntpd.sh
		-->users.sh

We will first go through all the configuration files in brief, and make the changes that are required for the setup. **The current script has all the defaults pre-setup and doesn't require any changes, unless specifically required.** 

## common.conf

This is an imporant configuration file and might require a few changes. This configuration file captures the following:

* Cluster Name
* ssh credentials (used for accessing other nodes by Cloudera Manager)
* Cloudera Manager Configuration (including for Kereberization)
* Parcel Locations for CDSW, Spark2 On Yarn, Cloudera Manager, CDH, Anaconda
* Roles that will be installed on CDH (HDFS, YARN, CDSW, SPARK2_ON_YARN, HIVE, OOZIE, IMPALA & HUE)
* Parameterized Variables for # of instances and instance types that will be setup for CDH Master & Worker, CDSW Master and Worker Nodes
* Location of post create/bootstrap scripts that need to be executed

Since this file has been largely parameterized, there are not many changes required here, unless you need to change the version of CDH, or if you get any errors on parcel locations or a specific version not be found

**Update 14th Jan, 2019**
There were some errors that I encountererd with Anaconda 5.0.1 not being available so I have made changes so that it now looks for 5.1.0.1

The following version are installed:
CDH: 5-14
SPARK: 2.2.0.cloudera2
Anaconda: 5.1.0.1
CDSW: 1.4.0.p1.431664


## aws.conf
This file just has links for the main conf files in the /aws folder. **No changes required**

## /aws/aws.conf
Serves as an include file. **No Changes Required**

## /aws/cluster.configs.HDFS.core_site_safety_valve.conf
A few standard configuration setting on block size etc. **No changes required**

## /aws/instances.conf
This file has the instance configuration options for CDH Master & Worker Nodes, CDSW Master & Worker Nodes. 

All the settings have been parameterized, so you can configure them from provider.properties. This file configures the intance type, instance name for CDH and additional properties such as rootVolumeSize, ebsVolumes for cdsw master and workers. For the most bit, you wouldn't need to tinker with this file much.

## /aws/kerberos.properties
This is an important file, and relevant ONLY IF you are kerberizing your cluster. If you don't want kerberization, simply remove this file from your folder before you upload the diretory to the director instance. If you are looking to bring up a test cluster with CDSW, no changes required. 

SECURITY_REALM:
HADOOPSECURITY.LOCAL

Users:
cm/admin@HADOOPSECURITY.LOCAL
cdsw@HADOOPSECURITY.LOCAL

KDC_HOST_IP: 0.0.0.0

**Update 16 Jan** I have added a script, which will automatically update the KDC_HOST_IP to the **internal** ip of the node where KDC is installed (instance currently running Director)

## /aws/provider.conf
Sets up aws properties in variables (accessKeyId etc.) **No changes required**

## /aws/provider.properties
This is an important file as various defaults are configured in this file. The important ones being as follows. **REMEMBER TO UPDATE** the accesskeyid and other fields before you deploy. 

		AMI_ID=ami-78485818
		AWS_ACCESS_KEY_ID=##Your AWS Key##
		AWS_REGION=us-west-1
		OWNER=rajat
		SECURITY_GROUP_ID=##Your Security Group ID##
		SUBNET_ID=##Your subnet ID##
		CDHMASTER_INSTANCE=c4.2xlarge
		CDHWORKER_INSTANCE=c4.4xlarge
		CDHWORKER_NODECOUNT=3
		CDSWWORKER_INSTANCE=c4.4xlarge
		CDSWMASTER_INSTANCE=c4.4xlarge
		CDSWWORKER_NODECOUNT=2
		YARN_RAM=4096
		YARN_VCORES=2
		NAME_PREFIX=rajat
		
Make sure you provide adequate nodes for CDH and CDSW workers as well as the appropriate shape for the instances. I typically assign larger shapes to CDSW Master and Workers (c4.8xlarge) for workshops. 

## SECRET.properties
You need to create this file in the same folder. For some reason, Github does not let me upload this file, perhaps due to the name. This file would have only one field. **Be mindful if you maintain your files on github; This file being shared publicly (and AWS ACCESS KEY) may give others access to your aws account.**

		AWS_SECRET_ACCESS_KEY=##Your AWS Key##

## /aws/ssh.properties
This file just has the default ssh username. In our case, that is ```centos```.You would also need to update the field:

		SSH_PRIVATEKEY=aws/cdsw-admin
		
Before you upload the zip file (having all properties files, scripts as mentioned in the steps below), please ensure that your public key and private key files are available in the ```/aws``` folder.
 
Your /aws folder should have the following files:

![Folder Struture](./images/folder-01.jpg) 


For GCP you will need to ensure that the plugin supports rhel7. Do this
by adding the following line to your =google.conf= file. This file
should be located in the provider directory:
=/var/lib/cloudera-director-plugins/google-provider-*/etc= (where the
=*= matches the version - something like =1.0.4= - of your plugins). You
will likely have to create your own copy of google.conf by copying
=google.conf.example= located in the same directory. Note that the exact
path to the relevant image is obtained by navigating to GCP's 'Images'
section and finding the corresponding OS/URL pair.

#+BEGIN_EXAMPLE
         rhel7 = "https://www.googleapis.com/compute/v1/projects/rhel-cloud/global/images/rhel-7-v20171025"
#+END_EXAMPLE

Assuming gcloud is on your path in that director instance, then this
script will do exactly what you need:

#+BEGIN_EXAMPLE
    sudo tee /var/lib/cloudera-director-plugins/google-provider-*/etc/google.conf 1>/dev/null <<EOF
    google {
      compute {
        imageAliases {
          centos6="$(gcloud compute images list  --filter='name ~ centos-6-v.*' --uri)",
          centos7="$(gcloud compute images list  --filter='name ~ centos-7-v.*' --uri)",
          rhel6="$(gcloud compute images list  --filter='name ~ rhel-6-v.*' --uri)",
          rhel7="$(gcloud compute images list  --filter='name ~ rhel-7-v.*' --uri)"
        }
      }
    }
    EOF
#+END_EXAMPLE

**** Azure
     :PROPERTIES:
     :CUSTOM_ID: azure
     :END:

Create a Cloudera Director by using the Microsoft Marketplace. In
keeping with minimizing what you have to do this project assumes you
have chosen the defaults whenever possible (e.g. networking etc)

You'll need to note: * the Resource Group that the director instance is
created in * The Region that the Resource Group is setup in * the public
domain name prefix of the director instance. (i.e. the hostname and
instance name of the director VM) * the host fqdn suffix (aka Private
DNS domain name). This is the DNS zone in which the Director and cluster
will be constructed. * the private IP address of the director instance
that is created (if you're going to put an MIT KDC on the Director
instance)

*** Installing MIT Kerberos (optional)
    :PROPERTIES:
    :CUSTOM_ID: installing-mit-kerberos
    :END:

If I choose to use MIT Kerberos I install the MIT KDC on the Director
VM, no matter which cloud provider I'm using.

I do that using
[[https://github.com/TobyHFerguson/director-scripts/blob/master/cloud-lab/scripts/install_mit_kdc.sh][install_mit_kdc.sh]]
to install an mit kdc. (There's also
[[https://github.com/TobyHFerguson/director-scripts/blob/master/cloud-lab/scripts/install_mit_client.sh][install_mit_client.sh]]
to create a client for testing purposes.).


** Preparation
   :PROPERTIES:
   :CUSTOM_ID: preparation
   :END:

- Choose the cloud provider you're going to work with and edit the
  =$PROVIDER/*.properties= and =$PROVIDER/SECRET= files appropriately.
- Ensure that all the files (including the SSH key file) is available to
  director (i.e copy or clone as necessary to the director server
  machine).
- Ensure that the [[SECRET files]] are in place
- Ensure that the =$PROVIDER/kerberos.properties= file is either absent
  (you don't want a kerberized cluster) or is present and correct (in
  particular you want to ensure that the =KDC_HOST_IP= property is set
  to the /ip address/ of the KDC server host (which should also be the
  Director host). Note that its the /ip address/ that you should use
  here because of a CDSW/Kubernetes defect: [[https://jira.cloudera.com/browse/DSE-1796][DSE-1796]]

** Cluster Creation
   :PROPERTIES:
   :CUSTOM_ID: cluster-creation
   :END:

- Execute a director bootstrap command using the cloud provider you
  chose, but make sure you do it from the top directory (i.e. the one where the =common.conf= file is located).
  
  
  
  
  
  
  

#+BEGIN_SRC sh
    cloudera-director bootstrap-remote $PROVIDER.conf --lp.remote.username=admin --lp.remote.password=admin
#+END_SRC

See [[No provider]] for what happens if you're in the wrong directory.

** Post Creation
   :PROPERTIES:
   :CUSTOM_ID: post-creation
   :END:

- Once completed, use your cloud provider's console to find the public
  IP (e.g. =104.92.37.53=) address of the CDSW instance. Its name in the cloud
  provider's console will begin with =cdsw-=.
- You can reach the CDSW at =cdsw.104.92.37.53.nip.io=. See [[NIP.io tricks]] for
  details about how that =nip.io= stuff works.

All nodes in the cluster will contain the user =cdsw=. That user's
password is =Cloudera1=. (If you used my mit kdc installation scripts
from below then you'll also find that this user's kerberos username and
password are =cdsw= and =Cloudera1= also).

* Project Structure
** File Organization
   :PROPERTIES:
   :CUSTOM_ID: file-organization
   :END:

*** Overview
    :PROPERTIES:
    :CUSTOM_ID: overview
    :END:

The system comprises a set of files, some common across cloud providers,
and some specific to a particular cloud provider. The common files (and
those which indicate which cloud provider to user) are all in the top
level directory; the cloud provider specific files are cloud provider
specific directories.

*** File Kinds
    :PROPERTIES:
    :CUSTOM_ID: file-kinds
    :END:

There are three kinds of files:

- Property Files - You are expected to modify these. They match the
  =*.properties= shell pattern and use the
  [[https://docs.oracle.com/javase/8/docs/api/java/util/Properties.html#load-java.io.Reader-][Java
  Properties format]]
- Conf files - You are not expected to modify these. They match the
  =*.conf= shell pattern and use the
  [[https://github.com/typesafehub/config/blob/master/HOCON.md][HOCON
  format]] (a superset of JSON).
- SECRET files - these have the prefix =SECRET= and are used to hold
  secrets for each provider. The exact format is provider specific.

The intent is that those items that you need to edit are in a format
(i.e. =*.properties= files) that is easy to edit, whereas those items
that you don't need to touch are in the harder to edit HOCON format
(i.e. =*.conf= files).

*** Directory Structure
    :PROPERTIES:
    :CUSTOM_ID: directory-structure
    :END:

The top level directory contains the main =conf= files (=aws.conf=,
=azure.conf= & =gcp.conf=). These are the files that indicate which
cloud provider is to be used.

The =aws=, =azure= and =gcp= directories contain the files relevant to
each cloud provider. We'll reference the general notion of a provider
directory using the =$PROVIDER= nomenclature, where =$PROVIDER= takes
the value =aws=, =azure= or =gcp=.

The main configuration file is =$PROVIDER.conf=. This file itself
includes the files needed for the specific cloud provider. We will only
describe the properties files here:

- =$PROVIDER/provider.properties= - a file containing the provider
  configuration for Amazon Web Services
- =$PROVIDER/ssh.properties= - a file containing the details required to
  configure passwordless ssh access into the machines that director will
  create.
- =$PROVIDER/kerberos.properties= - an /optional/ file containing the
  details of the Kerberos Key Distribution Center (KDC) to be used for
  kerberos authentication. (See Kerberos Tricks below for details on how
  to easily setup an MIT KDC and use it). /If/ =kerberos.properties= is
  provided then a secure cluster is set up. If =kerberos.properties= is
  not provided then an insecure cluster will be setup.

** SECRET files
   :PROPERTIES:
   :CUSTOM_ID: secret-files
   :END:

SECRET files are ignored by GIT and you must construct them yourself. We
recommend setting their mode to 600, although that is not enforced
anywhere.

*** AWS
   :PROPERTIES:
   :CUSTOM_ID: aws
   :END:

The secret file for AWS is =aws/SECRET.properties=. It is in Java
Properties format and contains the AWS secret access key:

#+BEGIN_EXAMPLE
    AWS_SECRET_ACCESS_KEY=
#+END_EXAMPLE

Mine, with dots hiding characters from the secret key, looks like:

#+BEGIN_EXAMPLE
    AWS_SECRET_ACCESS_KEY=53Hrd................r0wiBbKn3
#+END_EXAMPLE

If you fail to set up the =AWS_SECRET_KEY= then you'll find that
cloudera-director silently fails, but grepping for =AWS_SECRET_KEY= in
the local log file will reveal all:

#+BEGIN_SRC sh
    [centos@ip-10-0-0-239 ~]$ unset AWS_ACCESS_KEY_ID #just to make sure its undefined!
    [centos@ip-10-0-0-239 ~]$ cloudera-director bootstrap-remote filetest.conf --lp.remote.username=admin --lp.remote.password=admin
    Process logs can be found at /home/centos/.cloudera-director/logs/application.log
    Plugins will be loaded from /var/lib/cloudera-director-plugins
    Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=256M; support was removed in 8.0
    Cloudera Director 2.4.0 initializing ...
    [centos@ip-10-0-0-239 ~]$ 
#+END_SRC

Looks like its failed, right, because it doesn't continue on. No error
message! But if you execute:

#+BEGIN_EXAMPLE
    [centos@ip-10-0-0-239 ~]$ grep AWS_SECRET ~/.cloudera-director/logs/application.log
    com.typesafe.config.ConfigException$UnresolvedSubstitution: filetest.conf: 28: Could not resolve substitution to a value: ${AWS_SECRET_ACCESS_KEY}
#+END_EXAMPLE

You'll discover the problem! (Or there's another problem, and you should
look in that log file for details).

*** Azure
 The secet file for Azure is called =SECRET.properties=. It contains a
 single key value pair, where the key is =CLIENTSECRET=.

Here's my =azure/SECRET.properties= file:

#+BEGIN_EXAMPLE
CLIENTSECRET=jhwf4Gf+ ... zD+e3k=
#+END_EXAMPLE

*** GCP
    :PROPERTIES:
    :CUSTOM_ID: gcp
    :END:

The secret file for GCP is called =SECRET.json=. It contains the full
Google Secret Key, in JSON format, that you obtained when you made your
google account.

Mine, with characters of the private key id and lines of the private key
replaced by dots, looks like:

#+BEGIN_EXAMPLE
    {
      "type": "service_account",
      "project_id": "gcp-se",
      "private_key_id": "b27f..................66fea",
      "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQDMUKtOk000wkvJ\np/ZdwfkbpowUGMqpn2a0oQ9eTwIaLnPvrTIP3JcibWU7xkzoPXlD4hiANlkSqDqy
    .
    .
    .
    .
    .
    .
    UC2sMUZ1rtLCv14qg4iiXuA/RExTs1zRaZZ0r4c\nTDiZwBJEbs0flCAziv7mJ4TZ3LfGKCtrTOhUWRw/jfDHP+uJOpH2isGmytZ7uWVN\ndfllnxLITzHEQEMh0rbc/g3n\n-----END PRIVATE KEY-----\n",
      "client_email": "tobys-service-account@gcp-se.iam.gserviceaccount.com",
      "client_id": "108988546221753267035",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://accounts.google.com/o/oauth2/token",
      "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
      "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/tobys-service-account%40gcp-se.iam.gserviceaccount.com"
    }
#+END_EXAMPLE

* Troubleshooting
   :PROPERTIES:
   :CUSTOM_ID: troubleshooting
   :END:

There are two logs of interest:

- client log: $HOME/.cloudera-director/logs/application.log on client
  machine
- server log: /var/log/cloudera-director-server/application.log on
  server machine

If the cloudera-director client fails before communicating with the
server you should look in the client log. Otherwise look in the server
log.

The server log can be large - I /truncate/ it frequently (i.e. =echo >
/var/log/cloudera-director-server/application.log=) while the Director
server is running; especially before using a new conf file! Don't
simply delete it; doing so will mess up the Director (unless the
Director server is stopped)

** No provider
If you see this:

#+BEGIN_EXAMPLE
 * No provider configuration block found
#+END_EXAMPLE

then you've likely executed =cloudera-bootstrap= in the PROVIDER directory. You need to be in the top directory (where the =common.conf= file is) and execute =cloudera-bootstrap= there.
** GCP
    :PROPERTIES:
    :CUSTOM_ID: gcp-1
    :END:

*** No Plugin
     :PROPERTIES:
     :CUSTOM_ID: no-plugin
     :END:

If the client fails with this message:

#+BEGIN_SRC sh
    * ErrorInfo{code=PROVIDER_EXCEPTION, properties={message=Mapping for image alias 'rhel7' not found.}, causes=[]}
#+END_SRC

then you've not configured the plugin for GCP, as detailed in the
[[GCP Director Configuration]] section.

*** Old Plugin
     :PROPERTIES:
     :CUSTOM_ID: old-plugin
     :END:

If the client fails thus:

#+BEGIN_EXAMPLE
    * Requesting an instance for Cloudera Manager ............ done
    * Installing screen package (1/1) .... done
    * Suspended due to failure ...
#+END_EXAMPLE

and the server log contains something like this:

#+BEGIN_EXAMPLE
    peers certificate marked as not trusted by the user
#+END_EXAMPLE

then you've got a plugin configured, but its out of date. Update is,
as per the [[GCP Director Configuration]] section.

* Limitations & Issues
   :PROPERTIES:
   :CUSTOM_ID: limitations-issues
   :END:

Relies on [[NIP.io tricks]] to make it work.

Requires that the CDSW port be on the public internet.

* Appendix
** NIP.io tricks
   :PROPERTIES:
   :CUSTOM_ID: nip.io-tricks
   :END:

[[http://nip.io][NIP.io]] is a public bind server that uses the FQDN
given to return an address. A simple explanation is if you have your kdc
at IP address =10.3.4.6=, say, then you can refer to it as
=kdc.10.3.4.6.nip.io= and this name will be resolved to =10.3.4.6=
(indeed, =foo.10.3.4.6.nip.io= will likewise resolve to the same actual
IP address). (Note that earlier releases of this project used =xip.io=,
but that's located in Norway and for me in the USA =nip.io=, located in
the Eastern US, works better.)

This technique is used in two places: + In the director conf file to
specify the IP address of the KDC - instead of messing around with bind
or =/etc/hosts= in a bootstrap script etc. simply set the KDC\_HOST to
=kdc.A.B.C.D.xip.io= (choosing appropriate values for A, B, C & D as per
your setup) + When the cluster is built you will access the CDSW at the
public IP address of the CDSW instance. Lets assume that that address is
=C.D.S.W= (appropriate, some might say) - then the URL to access that
instance would be http://ec2.C.D.S.W.xip.io

This is great for hacking around with ephemeral devices such as VMs and
Cloud images!

** Useful Scripts
   :PROPERTIES:
   :CUSTOM_ID: useful-scripts
   :END:

*** Server Config
    :PROPERTIES:
    :CUSTOM_ID: server-config
    :END:

In =/var/kerberos/krb5kdc/kdc.conf=:

#+BEGIN_EXAMPLE
    [kdcdefaults]
     kdc_ports = 88
     kdc_tcp_ports = 88

    [realms]
     HADOOPSECURITY.LOCAL = {
     acl_file = /var/kerberos/krb5kdc/kadm5.acl
     dict_file = /usr/share/dict/words
     admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
     supported_enctypes = aes256-cts-hmac-sha1-96:normal aes128-cts-hmac-sha1-96:normal arcfour-hmac-md5:normal
     max_renewable_life = 7d
    }
#+END_EXAMPLE

In =/var/kerberos/krb5kdc/kadm5.acl= I setup any principal with the
=/admin= extension as having full rights:

#+BEGIN_EXAMPLE
    */admin@HADOOPSECURITY.LOCAL    *
#+END_EXAMPLE

I then execute the following to setup the users etc:

#+BEGIN_SRC sh
    sudo kdb5_util create -P Passw0rd!
    sudo kadmin.local addprinc -pw Passw0rd! cm/admin
    sudo kadmin.local addprinc -pw Cloudera1 cdsw

    systemctl start krb5kdc
    systemctl enable krb5kdc
    systemctl start kadmin
    systemctl enable kadmin
#+END_SRC

Note that the CM username and credentials are
=cm/admin@HADOOPSECURITY.LOCAL= and =Passw0rd!= respectively.

*** Client Config (Managed by Cloudera Manager)
    :PROPERTIES:
    :CUSTOM_ID: client-config-managed-by-cloudera-manager
    :END:

In =/etc/krb5.conf= I have this:

#+BEGIN_EXAMPLE
    [libdefaults]
     default_realm = HADOOPSECURITY.LOCAL
     dns_lookup_realm = false
     dns_lookup_kdc = false
     ticket_lifetime = 24h
     renew_lifetime = 7d
     forwardable = true
     default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
     default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5
     permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5

    [realms]
     HADOOPSECURITY.LOCAL = {
      kdc = 10.0.0.82
      admin_server = 10.0.0.82
     }
#+END_EXAMPLE

(Note that the IP address used is that of the private IP address of the
director server; this is stable over reboot so works well)

** ActiveDirectory
   :PROPERTIES:
   :CUSTOM_ID: activedirectory
   :END:

(Deprecated - I found this image to be unstable. It would just stop
working after 3 days or so.) I use a public ActiveDirectory ami setup by
Jeff Bean: =ami-a3daa0c6= to create an AD instance.

The username/password to the image are =Administrator/Passw0rd!=

Allow at least 5, maybe 10 minutes for the image to spin up and work
properly.

The kerberos settings (which you'd put into =kerberos.conf=) are:

#+BEGIN_EXAMPLE
    krbAdminUsername: "cm@HADOOPSECURITY.LOCAL"
    krbAdminPassword: "Passw0rd!
    KDC_TYPE: "Active Directory"
    KDC_HOST: "hadoop-ad.hadoopsecurity.local"
    KDC_HOST_IP: # WHATEVER THE INTERNAL IP ADDRESS IS FOR THIS INSTANCE
    SECURITY_REALM: "HADOOPSECURITY.LOCAL"
    AD_KDC_DOMAIN: "OU=hadoop,DC=hadoopsecurity,DC=local"
    KRB_MANAGE_KRB5_CONF: true
    KRB_ENC_TYPES: "aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arcfour-hmac-md5"
#+END_EXAMPLE

(Don't forget to drop the aes256 encryption if your images don't have
the Java Crypto Extensions installed)

** Standard users and groups
   :PROPERTIES:
   :CUSTOM_ID: standard-users-and-groups
   :END:

I use the following to create standard users and groups, running this on
each machine in the cluster:

#+BEGIN_SRC sh
    sudo groupadd supergroup
    sudo useradd -G supergroup -u 12354 hdfs_super
    sudo useradd -G supergroup -u 12345 cdsw
    echo Cloudera1 | sudo passwd --stdin cdsw
#+END_SRC

And then adding the corresponding hdfs directory from a single cluster
machine:

#+BEGIN_SRC sh
    kinit cdsw
    hdfs dfs -mkdir /user/cdsw
#+END_SRC
