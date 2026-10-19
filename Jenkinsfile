pipeline {
  agent any

  triggers {
    // Lets a GitHub webhook start this job
    githubPush()
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    APP_HOST = '10.0.0.34'
    APP_USER = 'opc'
    APP_DIR  = '/var/www/ci-lab'
    REPO_URL = 'https://github.com/abdulzafermobile/oci-lab1-static-site.git'
    SSH_OPTS = '-o StrictHostKeyChecking=accept-new -o BatchMode=yes -o ConnectTimeout=10'
  }

  stages {

    stage('Validate') {
      steps {
        // Fail BEFORE touching the server if the content is obviously wrong
        sh '''
          set -eu
          test -f public/index.html
          grep -q 'CI/CD Lab 1' public/index.html
          echo "Validated commit $GIT_COMMIT"
        '''
      }
    }

    stage('Deploy over SSH') {
      steps {
        sshagent(credentials: ['ssh-oci-vm-key']) {
          // Single-quoted Groovy string: Jenkins substitutes nothing, the shell does.
          // The remote script runs on the app VM. Its end marker REMOTE must start at column 0.
          sh '''
            set -eu
            ssh $SSH_OPTS "$APP_USER@$APP_HOST" bash -s -- "$APP_DIR" "$REPO_URL" "$GIT_COMMIT" "$BUILD_NUMBER" <<'REMOTE'
set -eu
APP_DIR="$1"; REPO_URL="$2"; COMMIT="$3"; BUILD="$4"
CONF=/etc/nginx/conf.d/ci-lab.conf

# First run: clone. Every run: move to EXACTLY the commit this build validated.
if [ ! -d "$APP_DIR/.git" ]; then
  git clone --depth 1 --branch main "$REPO_URL" "$APP_DIR"
fi
cd "$APP_DIR"
git fetch --depth 1 origin "$COMMIT"
git reset --hard FETCH_HEAD

# Stamp the deployment so anyone can see which build is live
echo "build $BUILD commit $COMMIT" > public/BUILD.txt

# Make sure SELinux labels the files as web content
sudo restorecon -R "$APP_DIR"

# Install the new config, but keep the old one in case validation fails
if [ -f "$CONF" ]; then sudo cp "$CONF" "$CONF.bak"; fi
sudo cp deploy/ci-lab.conf "$CONF"

# Validate BEFORE reloading. A bad config is rolled back and the site stays up.
if sudo nginx -t; then
  sudo systemctl reload nginx
else
  echo "Nginx config invalid - restoring the previous config"
  if [ -f "$CONF.bak" ]; then sudo cp "$CONF.bak" "$CONF"; else sudo rm -f "$CONF"; fi
  exit 1
fi
REMOTE
          '''
        }
      }
    }

    stage('Verify') {
      steps {
        sh '''
          set -eu
          curl -fsS --max-time 15 "http://$APP_HOST/" | grep -q 'CI/CD Lab 1'
          echo "Site content OK"
          curl -fsS --max-time 15 "http://$APP_HOST/BUILD.txt"
        '''
      }
    }
  }
}
