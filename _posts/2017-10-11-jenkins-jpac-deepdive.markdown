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
	
### MavenBuild.buildWithMaven
	#!/usr/bin/env groovy
	
	package com.optum.jenkins.pipeline.library.maven
	import com.cloudbees.groovy.cps.NonCPS
	import com.optum.jenkins.pipeline.library.utils.Constants
	import com.optum.jenkins.pipeline.library.scm.Git
	
	/**
	 * Runs a Maven build
	 * @param mavenGoals String Providing a way to override the default Maven Goals
	 * @param mavenOpts String Sets the environment variable MAVEN_OPTS
	 * @param mavenProfiles String Comma seperated list of Maven Profiles to pass into the command line.  Defaults to null.  Ex: j2ee1.6,sonar,
	 * @param javaVersion String The Java version to use. Defaults to Constants.JAVA_VERSION.
	 * @param mavenVersion String The Maven version to use. Defaults to Constants.MAVEN_VERSION
	 * @param jacocoMavenPluginVersion String The JaCoCo Maven plugin version. (See http://www.eclemma.org/jacoco/trunk/doc/maven.html)
	 * @param isDebugMode boolean Indicates if Maven should be run in debug (-X). Defaults to false.
	 * @param skipTests boolean Indicates if UnitTests should be skipped. Defaults to false.
	 * @param surefireReportsPath  Allows you to override the default surefireReportsPath
	 * @param uploadUnitTestResults Defaults to true.  Allows you to override the default
	 * @param uploadJacocoResults Defaults to true.  Allows you to override the default
	 * @param runJacocoCoverage boolean Indicates if JaCoCo Code Coverage should be run. Defaults to true.
	 * @param ignoreTestFailures boolean Indicates if Test Failures should be ignored. Defaults to true.  The following Sonar Scan will handle test failures.
	 * @param settingsXml Allows you to override the settings.xml on the command line so you can use one from your repo
	 * @param additionalProps Map An optional map of any additional properties that should be set.
	 */
	def buildWithMaven(body) {
	    def config = [
	       mavenGoals: null,  // defaulting to an invalid value so that it can be replaced later with values from the config
	       mavenOpts: "-Xmx1024m",
	       mavenProfiles: null,
	       javaVersion: Constants.JAVA_VERSION,
	       mavenVersion: Constants.MAVEN_VERSION,
	       jacocoMavenPluginVersion: Constants.JACOCO_MAVEN_PLUGIN_VERSION,
	       isDebugMode: false,
	       skipTests: false,
	       surefireReportsPath: "target/surefire-reports/*.xml",
	       uploadUnitTestResults: true,
	       uploadJacocoResults: true,
	       runJacocoCoverage: true,
	       ignoreTestFailures: true,
	       settingsXml: null,
	       additionalProps: null
	    ]
	    body.resolveStrategy = Closure.DELEGATE_FIRST
	    body.delegate = config
	    body()
	    
	    echo "buildWithMaven arguments: $config"
	    def mavenGoals
	    
	    // if the mavenGoals were not overwritten in the config, then use the defaults above in the config
	    if (config.mavenGoals) {
	    	
	    	// if you are passing in the mavenGoals, all other properties passed in are ignored.  You can pass in everything you need as part of these mavenGoals
	    	mavenGoals = config.mavenGoals
	    			
	    } else {
	    	
	    	// Building up the mavenGoals based upon the config passed in
	    	mavenGoals = "-U -e clean "
	    			
	    	if (config.runJacocoCoverage) {
	    		
	    		// if you are running JaCoCo code coverage then it is assumed that you want to run the Unit Tests.  The skipTests parameter is ignored.
	    		mavenGoals += "org.jacoco:jacoco-maven-plugin:${config.jacocoMavenPluginVersion}:prepare-agent install org.jacoco:jacoco-maven-plugin:${config.jacocoMavenPluginVersion}:report "
	        			
	    		if (config.ignoreTestFailures){
	    			mavenGoals += "-Dmaven.test.failure.ignore=true "
	    	    }		
	    	
	    	} else {
	    		
	    		echo "disabling JaCoCo code coverage"
	    		if (config.skipTests)  {
	    			
	    			// do not run JaCoCo Code Coverage and skip the Unit tests
	    			echo "disabling unit test execution"
	    			mavenGoals += "install -Dmaven.test.skip=true "
	    			
	    		} else {
	    			
	    			// do not run JaCoCo Code Coverage but run the unit tests
	    			mavenGoals += "install "	
		    		if (config.ignoreTestFailures){
		    			mavenGoals += "-Dmaven.test.failure.ignore=true "
		    	    }
	    		}
	    	}
	    	
	    	if (config.isDebugMode) {
	        	echo "enabling debug mode"
	        	mavenGoals += "-X "
	        }
	    	
	        if (config.additionalProps) {
	            for (def entry : config.additionalProps) {
	            	mavenGoals += "-D$entry.key=\"$entry.value\" "
	            }
	        }  
	        
	        if (config.mavenProfiles){
	
	            mavenGoals += "-P" + config.mavenProfiles + " "
	        }
	        
		if (config.settingsXml){
	    	    mavenGoals += "-s " + config.settingsXml + " "
		}
	    }
		
	    mavenGoals = "mvn " + mavenGoals
	
	    withEnv(["JAVA_VERSION=${config.javaVersion}", "MAVEN_VERSION=${config.mavenVersion}", "MAVEN_OPTS=${config.mavenOpts}"]) {
	        withEnv(["JAVA_HOME=${tool 'java'}", "MAVEN_HOME=${tool 'Maven'}"]) {
	            withEnv(["PATH=${JAVA_HOME}/bin:${MAVEN_HOME}/bin:$PATH"]) {        	
	            	command(mavenGoals)
	            }
	        }
	    }
	    
	    // publish junit results and JaCoCo code coverage results back to Jenkins
	    if (config.uploadUnitTestResults) {
	    	junit allowEmptyResults: true, testResults: config.surefireReportsPath
	    }
	    
	    // This step is dependent upon the JaCoCo Plugin being installed into your Jenkins Instance.
	    try {
		    if (config.uploadJacocoResults) {
		    	echo "uploading the JaCoCo results into Jenkins"
		    	step( [ $class: 'JacocoPublisher' ] )
		    }
	    }
	    catch(Exception ex){
	    	error("You either need to set 'uploadJacocoResults = false' in your config that you are passing in or you need to install the JaCoCo plugin into your Jenkins Instance")
	    }
	}






