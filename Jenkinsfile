/**********************************************************************
 *   Parameters and settings
 **********************************************************************/

MAX_AGE=getenv("MAX_AGE", "1M")
KEEP_INC=getenv("KEEP_INC", "2")
FULLAGE=getenv("FULL_AGE", "7D")
PREFIX=getenv("PREFIX", "autobackup")
LABEL=getenv("LABEL", "autobackup")
CREDENTIALS_ID=getenv("CREDENTIALS_ID", "techops")
// parameter NODES is used to find the nodes on which to run.  if
// NODES is not defined it will introspect all nodes with label LABEL


nodes = nodesForLabel(LABEL)

try {
  nodes.each {
    node(it) {
      containers = sh( returnStdout:true, script:"docker ps -f label=auto.backup --format {{.Names}}").trim().split('\n')

      containers.each {
         backup(it.trim())
      }
    }
  }
}
catch(exc) {
  if (env.NOTIFY) {
    mail   to: env.NOTIFY,
      subject: "Failed: ${currentBuild.fullDisplayName}",
         body: "Backup Failed.  See ${env.BUILD_URL}"
    throw exc
  }
}

def backup(container) {

  echo "Looking up backup name"
  backupName = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup\"}}'").trim()

  volumes = getVolumesFrom(container)

  stage("Backup\n${backupName}") {
    volumes.each {
      incontainer container, {
	if ( sh( returnStatus: true, script: "test -d '${it}'" ) == 0 ) {
	  echo "Backing up ${it}"
	  duplicity(backupName, it)
	}
       }
     }
   }

  stage("Purge old\n${backupName}") {
    volumes.each {
      incontainer container, {
	if ( sh( returnStatus: true, script: "test -d '${it}'" ) == 0 ) {
	  echo "Purging up ${it}"
	  duplicity(backupName, it, "remove-older-than ${MAX_AGE} --force")
	  duplicity(backupName, it, "remove-all-inc-of-but-n-full ${KEEP_INC} --force")
	  duplicity(backupName, it, "cleanup --force")

	  echo "Collecting backup status"

	  volumetext=it.replaceAll('[^a-zA-Z0-9_]', '_')
	  
          sh "duplicity collection-status --s3-use-new-style s3+http://${env.BUCKET}/${PREFIX}/${backupName}${it} > ${backupName}-${volumetext}-summary.txt"
	  archive "${backupName}-${volumetext}-summary.txt"

	}
       }
     }
   }
}

def getVolumesFrom(container) {
  echo "Finding volumes in ${container}"
  volumeList = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup.volumes\"}}'").trim()

  if ( volumeList == "") {
    volumes = sh(returnStdout:true, script:"docker inspect -f '{{  range .Mounts }} {{.Destination}}{{end}}' ${container}").trim().split(' ')
  }
  else {
    volumes = volumeList.split(' ')
  }
  volumes
}

def duplicity(backupName, volume, op="") {
    if (op == "") {
      sh """duplicity     \
 	    --allow-source-mismatch \
 	    --no-encryption \
 	    --full-if-older-than ${FULLAGE} \
 	    --s3-use-new-style  ${volume} s3+http://${env.BUCKET}/${PREFIX}/${backupName}${volume}
"""
    }
    else {
      sh """duplicity     \
            ${op} \
 	    --allow-source-mismatch \
 	    --no-encryption \
 	    --s3-use-new-style s3+http://${env.BUCKET}/${PREFIX}/${backupName}${volume}
"""
      }
}

def nodesForLabel(labelToSelect) {
    if (env.NODES) {
      return env.NODES.split(" ")
    }

    listOfNodeNames = jenkins.model.Jenkins.instance.nodes.collect {
      node -> node.getLabelString().contains(labelToSelect) ? node.name : null
    }
    listOfNodeNames.removeAll(Collections.singleton(null))
    listOfNodeNames
}

def getenv(name, defaultValue) {
    if(getBinding().hasVariable(name)) {
       getBinding().getVariable(name)
    }

    defaultValue
}

/** Run the given BODY inside in a duplicity container with volumes
 * from CONTAINER mapped in.
 */

def incontainer(container, body) {
      docker.image("wernight/duplicity").inside("--volumes-from ${container} -v duplicity-cache:/home/duplicity/.cache/duplicity -v duplicity-gnupg:/home/duplicity/.gnupg --hostname autobackup") {
       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
				credentialsId: CREDENTIALS_ID, 
	     			accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
				secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
   	    body()
	 }
       }
     }

