pipeline {
  agent any
  environment {
    ORG = 'trilogy-group'
    APP_NAME = 'angular7-example-app'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
    DOCKER_REGISTRY_ORG = 'trilogy-group'
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        sh "wget -O /home/jenkins/gitleaks.sh -q https://raw.githubusercontent.com/trilogy-group/gitleaks-ci/master/gitleaks.sh && chmod +x /home/jenkins/gitleaks.sh && /home/jenkins/gitleaks.sh 1"
        container('jenkinsxio/jx:2.0.119')
        sh "npm install"
        sh "CI=true DISPLAY=:99 npm test"
        sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
        sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
        dir('./charts/preview') {
          sh "make preview"
          sh "jx preview --app $APP_NAME --dir ../.."
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        git 'https://github.com/trilogy-group/angular7-example-app.git'

        // so we can retrieve the version in later steps
        sh "echo \$(jx-release-version) > VERSION"
        sh "jx step tag --version \$(cat VERSION)"
        sh "git fetch --depth=2 -q && wget -O /home/jenkins/gitleaks.sh -q https://raw.githubusercontent.com/trilogy-group/gitleaks-ci/master/gitleaks.sh && chmod +x /home/jenkins/gitleaks.sh && /home/jenkins/gitleaks.sh 2"
        container('jenkinsxio/jx:2.0.119')
        sh "npm install"
        sh "CI=true DISPLAY=:99 npm test"
        sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
        sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"

        // Packaging artifacts as a tar file
        sh "cd .. && tar --exclude=\$(basename \$OLDPWD)/.git -cvzf \$REPO_OWNER-\$REPO_NAME-\$BRANCH_NAME-\$BUILD_NUMBER.tar.gz \$(basename \$OLDPWD) && cd \$(basename \$OLDPWD)"

        // Upload to Nexus
        sh "cd .. && curl -v -u \$(jx step credential --name \$REPO_OWNER-\$REPO_NAME -k username -n jx-staging):\$(jx step credential --name \$REPO_OWNER-\$REPO_NAME -k password -n jx-staging) --upload-file \$REPO_OWNER-\$REPO_NAME-\$BRANCH_NAME-\$BUILD_NUMBER.tar.gz \$(jx step credential --name \$REPO_OWNER-\$REPO_NAME -k endpoint -n jx-staging)/repository/\$(jx step credential --name \$REPO_OWNER-\$REPO_NAME -k repo -n jx-staging)-release-raw/\$REPO_OWNER-\$REPO_NAME-\$BRANCH_NAME-\$BUILD_NUMBER.tar.gz"
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        dir('./charts/angular7-example-app') {
          sh "jx step changelog --batch-mode --version v\$(cat ../../VERSION)"

          // release the helm chart
          sh "jx step helm release"

          // promote through all 'Auto' promotion Environments
          sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
        }
      }
    }
  }
}
