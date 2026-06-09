pipeline {
  agent {
    kubernetes {
      label 'sparkplug-agent-pod'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: sparkplug-build
    image: cirruslink/sparkplug-build:latest
    command:
    - cat
    tty: true
    resources:
      limits:
        memory: "4Gi"
        cpu: "1"
      requests:
        memory: "4Gi"
        cpu: "1"
  - name: jnlp
    volumeMounts:
    - mountPath: "/home/jenkins/.gnupg"
      name: "jenkins-home-gnupg"
      readOnly: false
  volumes:
  - name: "jenkins-home-gnupg"
    emptyDir:
      medium: ""
"""
    }
  }

  stages {

    stage('get-version') {
      steps {
        container('sparkplug-build') {
          script {
            def projectVersion = sh(
              script: "GRADLE_USER_HOME=\"/home/jenkins/.gradle\" ./gradlew properties -q | grep '^version:' | awk '{print \$2}'",
              returnStdout: true
            ).trim()

            echo "The project version is: ${projectVersion}"
            env.PROJECT_VERSION = projectVersion
          }
        }
      }
    }

    stage('build') {
      steps {
        container('sparkplug-build') {
          sh 'Xvfb :0 -screen 0 1600x1200x16 & export DISPLAY=:0'
          sh 'GRADLE_USER_HOME="/home/jenkins/.gradle" ./gradlew -Dorg.gradle.jvmargs="-Xmx1536m -Xms64m -Dfile.encoding=UTF-8 -Djava.awt.headless=true" clean build'
        }
      }
    }

    stage('sign') {
      steps {
        script {
        withCredentials([
          [$class: 'FileBinding', credentialsId: 'secret-subkeys.asc', variable: 'KEYRING'],
          [$class: 'StringBinding', credentialsId: 'gpg-passphrase', variable: 'KEYRING_PASSPHRASE']
        ]) {
          sh '''
            curl -o tck/build/hivemq-extension/sparkplug-tck-${PROJECT_VERSION}-signed.jar -F file=@tck/build/hivemq-extension/sparkplug-tck-${PROJECT_VERSION}.jar https://cbi.eclipse.org/jarsigner/sign
            export GPG_TTY=/dev/console

            gpg --batch --import "${KEYRING}"
            for fpr in $(gpg --list-keys --with-colons  | awk -F: \'/fpr:/ {print $10}\' | sort -u); do echo -e "5\ny\n" |  gpg --batch --command-fd 0 --expert --edit-key ${fpr} trust; done

            mkdir tck/build/hivemq-extension/working_tmp
            cd tck/build/hivemq-extension/working_tmp
            unzip ../sparkplug-tck-${PROJECT_VERSION}.zip
            mv ../sparkplug-tck-${PROJECT_VERSION}-signed.jar sparkplug-tck/sparkplug-tck-${PROJECT_VERSION}.jar
            zip -r ../sparkplug-tck-${PROJECT_VERSION}.zip sparkplug-tck
            cd ..
            gpg -v --no-tty --passphrase "${KEYRING_PASSPHRASE}" -c --batch sparkplug-tck-${PROJECT_VERSION}.zip

            echo "no-tty" >> ~/.gnupg/gpg.conf
            gpg -vvv --no-permission-warning --output "sparkplug-tck-${PROJECT_VERSION}.zip.sig" --batch --yes --pinentry-mode=loopback --passphrase="${KEYRING_PASSPHRASE}" --no-tty --detach-sig sparkplug-tck-${PROJECT_VERSION}.zip
            cd ../../
            ./package.sh
            gpg -vvv --no-permission-warning --output "Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip.sig" --batch --yes --pinentry-mode=loopback --passphrase="${KEYRING_PASSPHRASE}" --no-tty --detach-sig Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip
            gpg -vvv --verify Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip.sig
          '''
        }
      }
      }
    }

    stage('upload') {
      steps {
        script {
          def projectVersion = env.PROJECT_VERSION
          if (projectVersion.contains("SNAPSHOT")) {
            echo "This is a snapshot build: ${env.PROJECT_VERSION}"
            sshagent(credentials: ['projects-storage.eclipse.org-bot-ssh']) {
              sh '''
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/sparkplug/snapshot/*${PROJECT_VERSION}*
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/sparkplug/snapshot
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/snapshot
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip.sig genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/snapshot/
              '''
            }
          } else if (projectVersion.contains("M")) {
            echo "This is a milestone build: ${env.PROJECT_VERSION}"
            sshagent(credentials: ['projects-storage.eclipse.org-bot-ssh']) {
              sh '''
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/sparkplug/milestone/*${PROJECT_VERSION}*
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/sparkplug/milestone
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/milestone
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip.sig genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/milestone/
              '''
            }
         }  else {
            echo "This is a release build: ${env.PROJECT_VERSION}"
            sshagent(credentials: ['projects-storage.eclipse.org-bot-ssh']) {
              sh '''
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/sparkplug/release/*${PROJECT_VERSION}*
                ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes genie.sparkplug@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/sparkplug/release
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/release
                scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o BatchMode=yes tck/Eclipse-Sparkplug-TCK-${PROJECT_VERSION}.zip.sig genie.sparkplug@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/sparkplug/release/
              '''
            }
          }
        }
      }
    }
  }
}
