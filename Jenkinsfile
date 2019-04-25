// Set your project Prefix
def prefix      = "user11"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "user11-tasks-dev"
def prodProject = "user11-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        git credentialsId: '1cffd564-f9c7-468d-b56b-1963603a98cd', url: 'http://gogs-user11-nexus.apps.de98.openshift.opentlc.com/CICDLabs/openshift-tasks-private.git'

       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0.0"
          devTag  = "${version}-${currentBuild.number}" 

          // Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = "${version}"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"

        sh "${mvnCmd} clean package -DskipTests"

      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"

        sh "${mvnCmd} clean test"
      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"

        sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube-user11-nexus.apps.de98.openshift.opentlc.com/ -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag} -DskipTests"

      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"

        sh "${mvnCmd} clean deploy -DaltDeploymentRepository=nexus::default::http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases -DskipTests"

      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"

        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace: /tmp/workspace/Tasks/target/openshift-tasks.war).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus3.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"
        //sh "oc start-build tasks --from-file=/tmp/workspace/Tasks/target/openshift-tasks.war -n ${devProject}"

        script{ openshift.withCluster() { openshift.withProject("${devProject}") {
          openshift.selector("bc", "tasks").startBuild("--from-file=./target/openshift-tasks.war", "--wait=true")
          openshift.tag("tasks:latest", "tasks:${devTag}")
        }}}

        // TBD: Tag the image using the devTag.

      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"

        // TBD: Deploy the image
        // 1. Update the image on the dev deployment config
        // 2. Update the config maps with the potentially changed properties files
       // 3. Reeploy the dev deployment
       // 4. Wait until the deployment is running
       //    The following code will accomplish that by
       //    comparing the requested replicas
       //    (rc.spec.replicas) with the running replicas
       //    (rc.status.readyReplicas)
       //

      // def dc = openshift.selector("dc", "tasks").object()
      // def dc_version = dc.status.latestVersion
      // def rc = openshift.selector("rc", "tasks-${dc_version}").object()

        script{ openshift.withCluster() { openshift.withProject("${devProject}") {

          openshift.set("image", "dc/tasks", "tasks=docker-registry.default.svc:5000/${devProject}/tasks:${devTag}")

          openshift.selector('configmap', 'tasks-config').delete("--ignore-not-found=true")
          openshift.create("configmap", "tasks-config", "--from-file=application-users.properties=./configuration/application-users.properties", "--from-file=application-roles.properties=./configuration/application-roles.properties")

          openshift.selector("dc", "tasks").rollout().latest()

          def dc = openshift.selector("dc", "tasks").object()
          def dc_version = dc.status.latestVersion
          def rc = openshift.selector("rc", "tasks-${dc_version}").object()

          echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
          while (rc.spec.replicas != rc.status.readyReplicas) {
            sleep 5
            rc = openshift.selector("rc", "tasks-${dc_version}").object()
	  }
        }}}

      }
    }

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps {
        echo "Running Integration Tests"
        script {
          def status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          // The next bit works - but only after the application
          // has been deployed successfully
          status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
          echo "Status: " + status
          if (status != "201") {
              error 'Integration Create Test Failed!'
          }

      echo "Retrieving tasks"
      status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Accept: application/json' -X GET http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
      if (status != "200") {
          error 'Integration Get Test Failed!'
      }

      echo "Deleting tasks"
      status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -X DELETE http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/1").trim()
      if (status != "204") {
          error 'Integration Delete Test Failed!'
      }

        }
      }
    }

    // Copy Image to Nexus Docker Registry
    stage('Copy Image to Nexus Docker Registry') {
      steps {
        echo "Copy image to Nexus Docker Registry"
        sh "skopeo copy --src-tls-verify=false --dest-tls-verify=false --src-creds openshift:\$(oc whoami -t) --dest-creds admin:admin123 docker://docker-registry-default.apps.de98.openshift.opentlc.com/${devProject}/tasks:${devTag} docker://nexus-registry-${prefix}-nexus.apps.de98.openshift.opentlc.com/tasks:${devTag}"

        // Tag the built image with the production tag.
        script{ openshift.withCluster() {
          openshift.withProject("${prodProject}") {
            openshift.tag("${devProject}/tasks:${devTag}", "${devProject}/tasks:${prodTag}")
          }
        }}
      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"

        // TBD: 1. Determine which application is active
        //      2. Update the image for the other application
        //      3. Deploy into the other application
        //      4. Update Config maps for other application
        //      5. Wait until application is running
        //         See above for example code

        script{ openshift.withCluster() { openshift.withProject("${prodProject}") {
          def route = openshift.selector("route", "tasks").object()
          def previousApp = "${route.spec.to.name}"
          echo "Current app in use by route: ${previousApp}"

          def newApp = "tasks-blue"
          if (previousApp == "tasks-blue") {
            newApp = "tasks-green"
          }
          echo "App for new version of the project: ${newApp}"

          openshift.set("image", "dc/${newApp}", "${newApp}=docker-registry.default.svc:5000/${devProject}/tasks:${devTag}")

          openshift.selector('configmap', "${newApp}-config").delete("--ignore-not-found=true")
          openshift.create("configmap", "${newApp}-config", "--from-file=application-users.properties=./configuration/application-users.properties", "--from-file=application-roles.properties=./configuration/application-roles.properties")

          openshift.selector("dc", "${newApp}").rollout().latest()

          def dc = openshift.selector("dc", "${newApp}").object()
          def dc_version = dc.status.latestVersion
          def rc = openshift.selector("rc", "${newApp}-${dc_version}").object()

          echo "Waiting for ReplicationController ${newApp}-${dc_version} to be ready"
          while (rc.spec.replicas != rc.status.readyReplicas) {
            sleep 5
            rc = openshift.selector("rc", "${newApp}-${dc_version}").object()
	  }
        }}}
      }
    }

    stage('Switch over to new Version') {
      steps {

        input 'Proceed with Production deployment?'

        echo "Executing production switch"
        
        script{ openshift.withCluster() { openshift.withProject("${devProject}") {
          openshift.set("route-backends", "tasks", "${newApp}=100", "${previousApp}=0")
        }}}

      }
    }
  }
}

