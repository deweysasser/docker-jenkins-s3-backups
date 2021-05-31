ARCHIVED
========

This project is no longer maintained.  I now prefer [restic](https://restic.net/) as a backup solution.

If someone would like to adopt it I'll be happy to transfer it.

---



Docker-Jenkins-S3 Backups
=========================

A [Jenkinsfile](http://jenkins-ci.org/) to drive backups on the
volumes of dynamically-discovered [docker](https://www.docker.com/)
images to backup volumes to an [S3](https://aws.amazon.com/s3/) bucket
using [duplicity](http://duplicity.nongnu.org/).

Who needs this?
---------------

You have docker containers running anywhere that can reach S3 and
want them backed up in a system driven by Jenkins.

Assumptions
-----------

You have a Jenkins system running with your docker hosts configured as
Jenkins build nodes.


Quick Start
-----------

* Put a docker label on docker container you want backed up with
  `auto.backup: NAME` where `NAME` is a simple ([a-zA-Z0-9_]+) and
  unique name.

* Create a Jenkins pipeline job on this pipeline with the following String
  parameters
  * `BUCKET` with the name of the S3 bucket
  * `NODES` with the list of Jenkins nodes on which to run
  * `CREDENTIALS_ID` with the ID of a set of Jenkins [AWS
  Credentials](https://wiki.jenkins.io/display/JENKINS/CloudBees+AWS+Credentials+Plugin) that can for read *and* write to the S3 bucket.

* run the pipeline on whatever schedule you decide.


Configuring the Docker containers for Backup
--------------------------------------------

Any docker container with the label `auto.backup` will be included in
the backup sets.  All volumes mounted in that container will be backed
up.

If you want only certain volumes backed up (or want other paths backed
up), add a docker label `auto.backup.volumes` with a space separated
list of paths to back up instead.

Configuring the Jenkins Job
---------------------------

See the next section for more dynamic configuration of NODES on which to run.

The following optional job parameters can also be defined:

* NOTIFY -- an email address to send job failure notifications

* MAX_AGE (default 1M) -- backups older than this amount will be pured
  (duplicity value for `remove-older-than` command)

* KEEP_INC (default 2) -- incremental backups will only be kept for
  this many full backups (duplicity value for
  `remove-all-inc-of-but-n-full`)

* FULLAGE (default "7D") -- lenght of time for incremental backup
  series.  A full backup will be done if past this time (duplicity
  value for `--full-if-older-than`)

* PREFIX (default "autobackup") -- the "directory" within the S3
  bucket to store backups

* LABEL (default "autobackup") -- when dynamically discovering nodes
  on which to run, this value is used as the Jenkins label to indicate
  we should run on that node.

* CREDENTIALS_ID (default "techops") -- Jenkins Credential ID for AWS
  credentials to use for S3 read/write


Jenkins Node Discovery
----------------------

If you do *NOT* have a build parameter called `NODES`, the build job
will dynamically look up all nodes with the label named in parameter
LABEL (default: "autobackup").  In this case, you must go into your
Jenkins configuration "script approval" section and grant the
following signatures approval:

     field hudson.model.Slave name
     method hudson.model.AbstractCIBase getNodes
     method hudson.model.Node getLabelString
     staticMethod java.util.Collections singleton java.lang.Object
     staticMethod jenkins.model.Jenkins getInstance
