#!groovy

node {

  try {
    notifyBuild('STARTED')
    deleteDir()

    stage ("Checkout"){
      checkout scm
    }

    //stage ("Build Container"){
    //  sh "sudo docker image tag nginx:latest nginx:${env.BUILD_NUMBER}"
    //}

    stage ("Create Test instance"){
      // clone the docker volume for test purposes
      sh "sudo docker volume create -d netapp -o from=tfrm18kickerapp_vol_html_root --name tfrm18kickerapp_vol_html_root_${env.BUILD_NUMBER} "
      sh "sudo docker volume create -d netapp -o from=tfrm18kickerapp_vol_site_config --name tfrm18kickerapp_vol_site_config_${env.BUILD_NUMBER} "
	  sh "sudo docker volume create -d netapp -o from=tfrm18kickerapp_vol_mysql --name tfrm18kickerapp_vol_mysql_${env.BUILD_NUMBER} "
	  
      // start the MySQL with the cloned storage
	  sh "sudo docker run -d --name mysql_${env.BUILD_NUMBER} -v tfrm18kickerapp_vol_mysql_${env.BUILD_NUMBER}:/var/lib/mysql mysql "
		

      // start the application
      sh "sudo docker run -d --name nginx_${env.BUILD_NUMBER} -v tfrm18kickerapp_vol_html_root_${env.BUILD_NUMBER}:/code -v tfrm18kickerapp_vol_site_config_${env.BUILD_NUMBER}:/etc/nginx/conf.d/ --network=tfrm18kickerapp_default --link tfrm18kickerapp_php_1 -p 80 nginx:latest "
	  
    }

    stage ("Automated Test Cases"){
      // get the port the container used for this test instance; will fail if container was not able to be started
      sh "sudo docker port nginx_${env.BUILD_NUMBER} 80 | cut -d\':\' -f2 > user.out"
      TEST_PORT = readFile('user.out').trim()

      // give the container 5 seconds to initialize the web server
	  sh "sleep 5"

      // connect to the webapp and verify it listens and is connected to the db
      //
      // to get IP of jenkins host (which must be the same container host where dev instance runs)
      // we passed it as an environment variable when starting Jenkins.  Very fragile but there is
      // no other easy way without introducing service discovery of some sort
      echo "Check if webapp port is listening"
      sh "curl http://${env.DOCKER_HOST_IP}:${TEST_PORT}/v1/ping -o curl.out"
      sh "cat curl.out"
      sh "awk \'/true/{f=1} END{exit!f}\' curl.out"
      echo "<<<<<<<<<< Access this test build at http://${DOCKER_HOST_IP}:${TEST_PORT} >>>>>>>>>>"
    }

    def push = ""
    stage ("Manual Test & Approve Push to Production"){
      // Test instance is online.  Ask for approval to push to production.
      notifyBuild('APPROVAL-REQUIRED')
      push = input(
        id: 'push', message: 'Push to production?', parameters: [
          [$class: 'ChoiceParameterDefinition', choices: 'Yes\nNo', description: '', name: 'Select yes or no']
        ]
      )
    }

    stage ("Deploy Production"){
      if (push == "Yes") {
        // if going to production: tag, stop, remove, and start updated application
        notifyBuild('PUSHING-TO-PROD')
        sh "sudo docker tag nginx:${env.BUILD_NUMBER} nginx:prod"
        sh "sudo docker kill nginx_${env.BUILD_NUMBER}"
        sh "sudo docker rm nginx_${env.BUILD_NUMBER}"
        sh "sudo docker run -d --name webapp --link redis -p 80:80 webapp:latest"
      }
      else{
        notifyBuild('NOT-PUSHING-TO-PROD')
      }
    }
  } catch (e) {
    // If there was an exception thrown, the build failed
    currentBuild.result = "FAILURE"
    throw e
  } finally {

    // Always destroy the test instance
    try {
      stage ("Destroy Test"){
        // remove the containers
        sh "sudo docker stop mysql_${env.BUILD_NUMBER} nginx_${env.BUILD_NUMBER}"
        sh "sudo docker rm -v mysql_${env.BUILD_NUMBER} nginx_${env.BUILD_NUMBER}"
        // remove the volume clone
        sh "sudo docker volume rm tfrm18kickerapp_vol_html_root_${env.BUILD_NUMBER} tfrm18kickerapp_vol_mysql_${env.BUILD_NUMBER} tfrm18kickerapp_vol_site_config_${env.BUILD_NUMBER}"
      }
    } finally {

      // Always send notifications, even if we failed to destroy test instance
      notifyBuild(currentBuild.result)
    }
  }
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus =  buildStatus ?: 'SUCCESSFUL'

  // Default values
  def colorCode = '#FF0000'

  // https://wiki.jenkins-ci.org/display/JENKINS/Build+User+Vars+Plugin variables available inside this block
  wrap([$class: 'BuildUser']) {
  user = "${env.BUILD_USER}"
  }

  def summary = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' by ${user} (${env.BUILD_URL}console)"
  //def summary = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"

  // Override default values based on build status
  if (buildStatus == 'FAILURE') {
    colorCode = '#FF0000'
  } else if (buildStatus == 'SUCCESSFUL') {
    colorCode = '#00FF00'
  } else {
    colorCode = '#FFFF00'
  }

  // Send notifications
  // slackSend (color: colorCode, message: summary)
}
