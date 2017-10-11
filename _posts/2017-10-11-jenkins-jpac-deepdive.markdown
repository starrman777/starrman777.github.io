---
layout: post
title:  "Jenkins Pipeline as Code Deep Dive"
date:   2017-10-11 09:00:01 -0400
categories: jekyll
---

These Code Snippets were used as part of a presentation that Eric Starr gave Oct 11, 2017 at the [Triangle Jenkins Meetup](https://www.meetup.com/Raleigh-Jenkins-Area-Meetup/events/241684336/).

### Jenkinsfile0
    #!/usr/bin/env groovy
    // Selecting a known Jenkins Build Slave
    node('docker-maven-slave') {
        stage('Checkout') {
	        echo "This is the Checkout"
	    }
	    stage('Build') {
		    echo "This is the Build"
	    }
	    stage('Sonar'){
		    echo "This is the Sonar Scan"  
	    }
	    stage('Artifactory'){
		    echo "This is the Artifactory Deployment"
	    }
    }

### Jenkinsfile0-Finished
	#!/usr/bin/env groovy
	node('docker-maven-slave') {
		stage('Checkout') {
		   echo "This is the Checkout"
		   git 'https://github.optum.com/devops-engineering/demo-maven-app.git'
		}
		stage('Build') {
	            sh '''
		    	. /etc/profile.d/jenkins.sh
			export JAVA_HOME=/tools/java/jdk1.8.0
			export MAVEN_HOME=/tools/maven/apache-maven-3.3.9
			export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin
			echo "kicking off the maven build now"
	         	mvn -U -e clean org.jacoco:jacoco-maven-plugin:0.7.9:prepare-agent install org.jacoco:jacoco-maven-plugin:0.7.9:report -Dmaven.test.failure.ignore=true -Dci.env=
	            '''
		}
		stage('Sonar'){
		    echo "This is the Sonar Scan"
	            sh '''
		    	. /etc/profile.d/jenkins.sh
			export JAVA_HOME=/tools/java/jdk1.8.0
			export MAVEN_HOME=/tools/maven/apache-maven-3.3.9
			export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin
			echo "kicking off the maven build now"
			mvn -e org.sonarsource.scanner.maven:sonar-maven-plugin:3.3.0.603:sonar \
			-Dsonar.projectVersion=1.0-SNAPSHOT \
			-Dsonar.projectKey=testing-zPipeline0-demo-maven-app \
			-Dsonar.projectName=zPipeline0-demo-maven-app \
			-Dsonar.links.scm=https://github.optum.com/my-organization/demo-maven-app.git \
			-Dsonar.links.ci=https://jenkins.optum.com/my-organization/job/zPipeline0 \
			-Dsonar.projectName=DEVOPS-demo-maven-app \
			-Dsonar.host.url=http://sonar.optum.com \
			-Dsonar.login=[TOKEN GOES HERE]
	            '''
		   
		}
		stage('Artifactory'){
			echo "This is the Artifactory Deployment"
		}
	}

### Jenkinsfile1-Finished
	#!/usr/bin/env groovy
	
	@Library("com.optum.jenkins.pipeline.library@v0.1.16")
	import com.optum.jenkins.pipeline.library.scm.Git
	import com.optum.jenkins.pipeline.library.maven.MavenBuild
	import com.optum.jenkins.pipeline.library.sonar.Sonar
	
	node('docker-maven-slave') {
		Git git = new Git()
		MavenBuild build = new MavenBuild()
		Sonar sonar = new Sonar()
	
		stage('Checkout') {
		   git.checkout()
		}
		stage('Build') {
		   def buildConfig = {}
		   build.buildWithMaven buildConfig
		}
		stage('Sonar'){
		    def sonarConfig = {
		        productName = "DEVOPS"              
		        projectName = "demo-maven-app"      
		        gitUserCredentialsId = 'sonar_tech'
		    }
		    sonar.scanWithMaven sonarConfig
		}
	}
	
### Jenkinsfile-declarative
	#!/usr/bin/env groovy
	
	@Library("com.optum.jenkins.pipeline.library@master")
	import com.optum.jenkins.pipeline.library.maven.MavenBuild
	
	pipeline {
	    agent { label 'docker-maven-slave' }
	    stages {
	        stage('Build') {
	            steps {
	                script{
			    def buildConfig = {
			        javaVersion = '1.7.0'
		            }
				
			    MavenBuild build = new MavenBuild()
			    build.buildWithMaven buildConfig
			}
		    }
		}       
	    }
	    post {
	        always {
	            echo 'This will always run'
			emailext body:  "Build URL: ${BUILD_URL}", 
				 subject: "$currentBuild.currentResult-$JOB_NAME", 
				 to: 'eric.starr@optum.com'
	        }
	        success {
	            echo 'This will run only if successful'
	        }
	        failure {
	            echo 'This will run only if failed'
	        }
	        unstable {
	            echo 'This will run only if the run was marked as unstable'
	        }
	        changed {
	            echo 'This will run only if the state of the Pipeline has changed'
	            echo 'For example, if the Pipeline was previously failing but is now successful'
	
	        }
	    }
	}




