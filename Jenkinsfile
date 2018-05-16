pipeline {
  agent any
  options {
    ansiColor('xterm')
    timestamps()
    timeout(time: 15, unit: 'MINUTES')
    buildDiscarder(logRotator(artifactNumToKeepStr: '5', numToKeepStr: '20'))
    disableConcurrentBuilds()
    skipDefaultCheckout true
  }
  parameters {
    string( \
      name: 'projectName', \
      description: 'Project Name, i.e: MyProject')
    string( \
      name: 'notUsedSince', \
      description: 'Is # of days ago, since not used artifacts will be deleted, if empty will take 30')
    string( \
      name: 'createdBefore', \
      description: 'In # of days ago, if empty will take 30')
    booleanParam( \
      defaultValue: true, \
      description: 'When dry run is true the job will not delete any artifact', \
      name: 'dryRun')
  }
  environment {
        ARTIFACTORY = credentials('Artifactory')
  }
  stages {
    stage('clean_checkout') {
      steps {
        echo 'Cleaning workspace...'
        deleteDir()
        checkout scm
      }
    }
    stage('Define Variables') {
      steps {
        script {
          // parsing params
          projectName = params.projectName.trim() ?: ''
          int createdBefore = params.createdBefore?.trim() ? params.createdBefore.toInteger() : 30
          int notUsedSince = params.notUsedSince?.trim() ? params.createdBefore.toInteger() : 30
          dryRun = params.dryRun
          // validating params
          createdBefore = createdBefore >= 30 ? createdBefore : 30
          notUsedSince = notUsedSince >= 30 ? notUsedSince : 30
          // defining new vars
          date = new Date()
          println("Actual Date is: "+date)
          //transform createdBefore in milliseconds
          createdBeforeMill = date.minus(createdBefore).getTime()
          println("Getting artifacts created before "+date.minus(createdBefore)+" ("+createdBeforeMill+")")
          //transform notUsedSince in milliseconds
          notUsedSinceMill = date.minus(notUsedSince).getTime()
          println("Getting artifacts not used since "+date.minus(notUsedSince)+" ("+notUsedSinceMill+")")
          //Define Artifactory URL and repo Name
          //ARTIFACTORY_URL is an environment variable in jenkins with a value like i.e: http://serverIP:8081/artifactory
          searchUrl = "${ARTIFACTORY_URL}/api/search"
          // the repo from which we want to delete artifacts
          repoName = 'gradle-dev-local'
          //Updating job attributes
          currentBuild.description = projectName+" Cleaned from "+repoName
        }
      }
    }
    stage('Searching Artifacts') {
      steps {
        sh """
          curl -u $ARTIFACTORY_USR:$ARTIFACTORY_PSW -X GET -o list.json \"${searchUrl}/usage?notUsedSince=${notUsedSinceMill}&createdBefore=${createdBeforeMill}&repos=${repoName}\"
        """
        script{
          list = readJSON file: 'list.json'
        }
      }
    }
    stage('Deleting Artifacts') {
      steps {
        script {
          String finalUri = ''
          String uri = ''
          String msg = ''
          int count = 0
          def filter = '.*/'+projectName+'/.*(war|apk)$'
          //delete pattern is because in my case I wanted to delete all the directory, not only the specific artifact
          def deletePattern = '[^/]*(war|apk)$'
          println("List of artifacts to be deleted are: ")
          list.results.each {
            uri = it['uri']
            if (uri =~ filter ) {
              count += 1
              println("Deleting artifact:")
              println("Initial URI: "+uri)
              //remove "api/storage/" part and the filename from the URI to obtain the diretory URI
              //YOu may have to adapt the modification to your artifactory structure
              finalUri = uri.replace("api/storage/", "").replaceFirst(deletePattern,"")
              println("Final URI: "+finalUri)
              if (!dryRun) {
                sh """
                  curl -u $ARTIFACTORY_USR:$ARTIFACTORY_PSW -X DELETE \"${finalUri}\"
                """
              }
            }
          }
          msg = count.toString()+" artifacts match with the deleting pattern.\n"
          if (dryRun) {
            currentBuild.displayName += " DRY RUN"
            msg += "\nDRY RUN, no delete action was taken"
          }
          println(msg)
        }
      }
    }
  }
}