node {
    
    stage('preparation')  {
         def userInput = input(
        id: 'userInput', message: 'preparation',
                        parameters: [
                                    [$class: 'TextParameterDefinition', defaultValue: 'cicd', description: 'dev env', name: 'DEV_PROJECT'],
                                   [$class: 'TextParameterDefinition', defaultValue: 'test1', description: 'stage env', name: 'STAGE_PROJECT']
                                 ] )
                      devenv = userInput.DEV_PROJECT
                      stageenv = userInput.STAGE_PROJECT
                      input message: "Is the PARAMETERS are correctly set for deployment?", ok: "Promote" 
        
    }
      def mvnHome
      mvnHome = tool 'maven'
      stage('Fetch Code') { 
        	git changelog: false, credentialsId: 'b7552763e472555d9593ff49af979d3e7a879884', poll: false, url: 'https://github.com/sudhirkr448/boxfuse-sample-java-war-hello.git'
   }
      stage('Build') {
       	echo "in Build"
       sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean install"
   }    
      stage('Sonar') {
       echo "in sonar" 
   	sh "'${mvnHome}/bin/mvn' -X sonar:sonar -Dsonar.host.url=http://10.131.0.64:9000 -Dsonar.login=efa577e3d61f46c582d3680ded89992ffd3a95b8 '-fpom.xml' '-Dsonar.projectKey=cicd' '-Dsonar.language=java' '-Dsonar.sources=src'"
   }
   stage('Nexus'){
       	echo "in Nexus"
   	sh "curl -v -u admin:admin123 --upload-file /var/lib/jenkins/jobs/cicd/jobs/cicd-tasks-pipeline/workspace/target/hello-1.0.war  http://10.131.0.59:8081/repository/myapp1/"
   }
   stage('Build Image') {
             when {
        expression {
       	openshift.withCluster() {
       		openshift.withProject(env.DEV_PROJECT) {
     	return !openshift.selector("bc", "jboss").exists()
   	
           }
       }}}
             steps {
        script {
              	openshift.withCluster() {
       		openshift.withProject(env.DEV_PROJECT) {
       		    	openshift.newBuild("--name=jboss", "--image-stream=jboss-eap70-openshift:1.5", "--binary=true")
       		}}}}
       		    
   }    
      stage('Build Image with app') {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp /var/lib/jenkins/jobs/cicd/jobs/cicd-tasks-pipeline/workspace/target/hello-1.0.war oc-build/deployments/ROOT.war"                                
           openshift.withCluster() {
             openshift.withProject("${devenv}") {
               openshift.selector("bc", "jboss").startBuild("--from-dir=oc-build", "--wait=true")
             }
           }
      }
      stage('deploy to Dev') {
                              openshift.withCluster() {
          openshift.withProject("${devenv}") {
            if (openshift.selector('dc', 'jboss').exists()) {
              openshift.selector('dc', 'jboss').delete()
              openshift.selector('svc', 'jboss').delete()
              openshift.selector('route', 'jboss').delete()
            }

            def app = openshift.newApp("jboss:latest")
            app.narrow("svc").expose();
            def dc = openshift.selector("dc", "jboss")
         openshiftScale(namespace: 'cicd', deploymentConfig: 'jboss',replicaCount: '1')
         openshift.tag("${devenv}/jboss:latest", "${devenv}/jboss:cicd")
         }
   	}
   }
   stage('Promote to TEST?') {
         timeout(time:15, unit:'MINUTES') {
         input message: "Promote to test?", ok: "Promote"
         }
       openshift.withCluster() {
         def version = "test"
         openshift.tag("${devenv}/jboss:cicd", "${stageenv}/jboss:${version}")
         openshift.withProject(env.STAGE_PROJECT) {
            if (openshift.selector('dc', 'jboss').exists()) {
            openshift.selector('dc', 'jboss').delete()
            openshift.selector('svc', 'jboss').delete()
            openshift.selector('route', 'jboss').delete()
            }
          openshift.newApp("jboss:${version}").narrow("svc").expose()
          //openshiftScale(namespace: 'test', deploymentConfig: 'jboss',replicaCount: '1')
         }
   	}
   }
}
