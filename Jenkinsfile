/**********************************************************************
 *   Parameters and settings
 **********************************************************************/

MAX_AGE=getenv("MAX_AGE", "1M")
KEEP_INC=getenv("KEEP_INC", "2")
FULLAGE=getenv("FULL_AGE", "7D")
PREFIX=getenv("PREFIX", "autobackup/restic")
LABEL=getenv("LABEL", "autobackup")
CREDENTIALS_ID=getenv("CREDENTIALS_ID", "techops")
BACKUP_PASSWORD=getenv("BACKUP_PASSWORD", "backup")
// parameter NODES is used to find the nodes on which to run.  if
// NODES is not defined it will introspect all nodes with label LABEL


nodes = nodesForLabel(LABEL)

try {
    nodes.each {
        node(it) {
            containers = sh( returnStdout:true, script:"docker ps -f label=auto.backup --format {{.Names}}").trim().split('\n')

            echo "Containers are: ${containers}"
            
            containers.each {
                container= it.trim()
                echo "Checking ${container} for MYSQL password"
                mysql_pass = sh(returnStdout:true, script:"docker inspect --format '{{.Config.Env}}' ${container} |  tr ' ' '\n' | grep MYSQL_ROOT_PASSWORD | sed 's/^.*=//'").trim()

                if (mysql_pass != "") {
                    // It must be mysql, so back it up that way
                    resticBackupMysql(it.trim(), mysql_pass)
                }
                else {
                    resticBackup(it.trim())
                }
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

def duplicityBackup(container) {

  echo "Looking up backup name"
  backupName = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup\"}}'").trim()

  volumes = getVolumesFrom(container)

  stage("Backup\n${backupName}") {
    volumes.each {
      inDuplicity container, {
	if ( sh( returnStatus: true, script: "test -d '${it}'" ) == 0 ) {
	  echo "Backing up ${it}"
	  duplicity(backupName, it)
	}
       }
     }
   }

  stage("Purge old\n${backupName}") {
    volumes.each {
      inDuplicity container, {
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

def resticBackup(container) {

    echo "Looking up backup name"
    backupName = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup\"}}'").trim()
    sh "rm -f *.txt"

    volumes = getVolumesFrom(container)
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
		      credentialsId: CREDENTIALS_ID, 
	     	      accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
		      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {

        targets=volumes.join(" ")

        stage("Backup\n${backupName}") {
            echo "Backing up ${backupName}:${it}"
            restic backupName, container, "init", true
            restic backupName, container, "backup ${targets} >${backupName}-backup.txt"
        }

        stage("Purge old\n${backupName}") {
	    echo "Purging old ${backupName}"
            restic backupName, container, "snapshots > ${backupName}-snapshots.txt"
            restic backupName, container, "forget --prune -d 7 -w 2 -m 6 -y 7 > ${backupName}-forget.txt"
        }
        
	archive "*.txt"
    }
}

def resticBackupMysql(container, password) {

    echo "Looking up backup name"
    backupName = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup\"}}'").trim()
    sh "rm -f *.txt"

    databases = getDatabasesFrom(container, password)
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
		      credentialsId: CREDENTIALS_ID, 
	     	      accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
		      secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {


        databases.each {

            stage("Backup DB\n${backupName} ${it}") {
                echo "Backing up ${backupName}:${it}"
                restic backupName, container, "init", true
                dumpToRestic backupName, container, it, password
            }
        }

        stage("Purge old\n${backupName}") {
	    echo "Purging old ${backupName}"
            restic backupName, container, "snapshots > ${backupName}-snapshots.txt"
            restic backupName, container, "forget --prune -d 7 -w 2 -m 6 -y 7 > ${backupName}-forget.txt"
        }
        
	archive "*.txt"
    }
}

def getDatabasesFrom(container, password) {
    echo "Finding databases in ${container}"
    dbList = sh(returnStdout:true, script:"docker ps -f name=${container} --format '{{.Label \"auto.backup.databases\"}}'").trim()

    if ( dbList == "") {
        databases = sh(returnStdout:true, script:"echo 'show databases;' | docker exec -i ${container} mysql -u root -p${password} | egrep -v 'Database|mysql|information_schema|performance_schema|sys'").split('\n')
    }
    else {
        databases = dbList.split(' ')
    }
    echo "Databases are: ${databases}"
    databases
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

def restic(backupName, container, text, status=false) {
    cmd= """docker run \
          --hostname autobackup \
          --rm \
          --volumes-from ${container} \
          -e 'RESTIC_PASSWORD=${BACKUP_PASSWORD}' \
          -e 'AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}' \
          -e 'AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}' \
          -e 'RESTIC_REPOSITORY=s3:s3.amazonaws.com/${env.BUCKET}/${PREFIX}/${backupName}' \
restic/restic \
${text}
"""
    echo "Running [${cmd}]"
    sh( returnStatus: status, script: cmd)
}

def dumpToRestic(backupName, container, database, password, status=false) {
    cmd= """ \
docker exec $container mysqldump -u root -p${password} ${database} | \
docker run -i \
          --hostname autobackup \
          --rm \
          -e 'RESTIC_PASSWORD=${BACKUP_PASSWORD}' \
          -e 'AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}' \
          -e 'AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}' \
          -e 'RESTIC_REPOSITORY=s3:s3.amazonaws.com/${env.BUCKET}/${PREFIX}/${backupName}' \
restic/restic \
backup --stdin --stdin-filename ${database}.sql 
"""
    echo "Running [${cmd}]"
    sh( returnStatus: status, script: cmd)
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

def inDuplicity(container, body) {
      docker.image("wernight/duplicity").inside("--volumes-from ${container} -v duplicity-cache:/home/duplicity/.cache/duplicity -v duplicity-gnupg:/home/duplicity/.gnupg --hostname autobackup") {
       withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
				credentialsId: CREDENTIALS_ID, 
	     			accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
				secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
   	    body()
	 }
       }
     }

/** Run the given BODY inside in a restic container with volumes
 * from CONTAINER mapped in.
 */

def inRestic(container, body) {
    docker.image("restic/restic").inside("--volumes-from ${container} --hostname autobackup") {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', 
			  credentialsId: CREDENTIALS_ID, 
	     		  accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
			  secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
   	    body()
	}
    }
}

