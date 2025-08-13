pipeline {
  agent any

  options {
    ansiColor('xterm')
    timestamps()
    durabilityHint('PERFORMANCE_OPTIMIZED')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    timeout(time: 60, unit: 'MINUTES')
    skipDefaultCheckout(true)
    // Fail fast on parallel stages
    parallelsAlwaysFailFast()
  }

  environment {
    // Tooling
    JAVA_HOME = "${tool 'jdk17'}"
    PATH = "${env.JAVA_HOME}/bin:${env.PATH}"

    // Android SDK (adjust if different on your agents)
    ANDROID_HOME = "/opt/android-sdk"
    ANDROID_SDK_ROOT = "/opt/android-sdk"
    GRADLE_OPTS = "-Dorg.gradle.daemon=false -Dorg.gradle.jvmargs='-Xmx2g -XX:+HeapDumpOnOutOfMemoryError'"

    // Sonar (scanner tool name must exist in Jenkins > Global Tool Config)
    SONAR_SCANNER = "${tool 'SonarQubeScanner'}"

    // Toggleable flags (can also be job parameters)
    RUN_DEP_CHECK = "true"
    RUN_GITLEAKS   = "true"
    RUN_SONAR      = "true"
    RUN_ANDROID_LINT = "true"
    RUN_TESTS      = "true"
    RUN_BUILD      = "true"
    RUN_FIREBASE_DEPLOY = "false"   // set "true" to enable Firebase App Distribution
    RUN_MOBSF      = "true"

    // MobSF endpoint (change to your URL)
    MOBSF_URL = "http://mobsf.local:8000"

    // Artifact/report dirs
    REPORT_DIR = "reports"
    BUILD_DIR  = "app/build"
  }

  parameters {
    booleanParam(name: 'RELEASE_BUILD', defaultValue: false, description: 'If true, assembleRelease; else assembleDebug')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm // honors job’s SCM & branch (good for PR/multibranch too)
        sh 'mkdir -p $REPORT_DIR'
      }
    }

    stage('Gradle wrapper integrity') {
      steps {
        sh 'chmod +x gradlew && ./gradlew --version'
      }
    }

    stage('Security & Quality Gates (Parallel)') {
      parallel {
        stage('SCA - OWASP Dependency-Check') {
          when { environment name: 'RUN_DEP_CHECK', value: 'true' }
          steps {
            sh '''
              mkdir -p $REPORT_DIR/dependency-check data/dependency-check
              docker run --rm \
                -v "$(pwd)":/src \
                -v "$(pwd)/data/dependency-check":/usr/share/dependency-check/data \
                -v "$(pwd)/$REPORT_DIR/dependency-check":/report \
                owasp/dependency-check:latest \
                --scan /src \
                --format "HTML" \
                --out /report \
                --project "AndroidApp"
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: "$REPORT_DIR/dependency-check/*", fingerprint: true, allowEmptyArchive: true
              publishHTML(target: [
                reportDir: "$REPORT_DIR/dependency-check",
                reportFiles: 'dependency-check-report.html',
                reportName: 'Dependency-Check'
              ])
            }
          }
        }

        stage('Secret Scanning - Gitleaks') {
          when { environment name: 'RUN_GITLEAKS', value: 'true' }
          steps {
            sh '''
              mkdir -p $REPORT_DIR/gitleaks
              docker run --rm -v "$(pwd)":/repo zricethezav/gitleaks:latest \
                detect --source=/repo \
                --report-format sarif \
                --report-path /repo/$REPORT_DIR/gitleaks/gitleaks.sarif
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: "$REPORT_DIR/gitleaks/*", fingerprint: true, allowEmptyArchive: true
            }
          }
        }

        stage('Android Lint') {
          when { environment name: 'RUN_ANDROID_LINT', value: 'true' }
          steps {
            sh '''
              ./gradlew --no-daemon lintDebug
              mkdir -p $REPORT_DIR/android-lint
              cp -f app/build/reports/lint-results*.* $REPORT_DIR/android-lint/ || true
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: "$REPORT_DIR/android-lint/*", fingerprint: true, allowEmptyArchive: true
              publishHTML(target: [
                reportDir: "$REPORT_DIR/android-lint",
                reportFiles: 'lint-results-debug.html',
                reportName: 'Android Lint'
              ])
            }
          }
        }

        stage('Unit Tests + Jacoco') {
          when { environment name: 'RUN_TESTS', value: 'true' }
          steps {
            sh '''
              ./gradlew --no-daemon testDebugUnitTest jacocoTestReport
              mkdir -p $REPORT_DIR/tests $REPORT_DIR/coverage
              cp -rf app/build/reports/tests/testDebugUnitTest/* $REPORT_DIR/tests/ || true
              cp -rf app/build/reports/jacoco/jacocoTestReport/* $REPORT_DIR/coverage/ || true
            '''
          }
          post {
            always {
              junit testResults: 'app/build/test-results/testDebugUnitTest/*.xml', allowEmptyResults: true
              publishHTML(target: [
                reportDir: "$REPORT_DIR/tests",
                reportFiles: 'index.html',
                reportName: 'Unit Tests'
              ])
              publishHTML(target: [
                reportDir: "$REPORT_DIR/coverage",
                reportFiles: 'index.html',
                reportName: 'Code Coverage (Jacoco)'
              ])
              archiveArtifacts artifacts: "$REPORT_DIR/tests/**,$REPORT_DIR/coverage/**", fingerprint: true, allowEmptyArchive: true
            }
          }
        }

        stage('SAST - SonarQube') {
          when { environment name: 'RUN_SONAR', value: 'true' }
          steps {
            withSonarQubeEnv('SonarQubeServer') {
              sh '''
                ${SONAR_SCANNER}/bin/sonar-scanner \
                  -Dsonar.projectKey=android-app \
                  -Dsonar.projectName=android-app \
                  -Dsonar.sources=app/src \
                  -Dsonar.java.binaries=app/build/intermediates/javac \
                  -Dsonar.kotlin.binaries=app/build/tmp/kotlin-classes \
                  -Dsonar.junit.reportPaths=app/build/test-results/testDebugUnitTest \
                  -Dsonar.java.coveragePlugin=jacoco \
                  -Dsonar.jacoco.reportPaths=app/build/jacoco/testDebugUnitTest.exec \
                  -Dsonar.androidLint.reportPaths=app/build/reports/lint-results-debug.xml
              '''
            }
          }
        }
      } // parallel
    }

    stage('SonarQube Quality Gate') {
      when { environment name: 'RUN_SONAR', value: 'true' }
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          script {
            def qg = waitForQualityGate() // requires SonarQube Scanner for Jenkins
            if (qg.status != 'OK') {
              error "Pipeline aborted due to SonarQube Quality Gate failure: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Build') {
      when { environment name: 'RUN_BUILD', value: 'true' }
      steps {
        sh '''
          if [ "${RELEASE_BUILD}" = "true" ]; then
            ./gradlew --no-daemon clean assembleRelease
          else
            ./gradlew --no-daemon clean assembleDebug
          fi
        '''
      }
      post {
        success {
          script {
            def apkPattern = params.RELEASE_BUILD ?
              "app/build/outputs/apk/release/*.apk" :
              "app/build/outputs/apk/debug/*.apk"
            archiveArtifacts artifacts: apkPattern, fingerprint: true
          }
        }
      }
    }

    stage('Deploy to Firebase (optional)') {
      when {
        allOf {
          environment name: 'RUN_FIREBASE_DEPLOY', value: 'true'
          environment name: 'RUN_BUILD', value: 'true'
        }
      }
      environment {
        // Supply either FIREBASE_TOKEN or GOOGLE_APPLICATION_CREDENTIALS (service account file)
        FIREBASE_APP_ID = '1:1234567890:android:abc123def456' // <-- change me
        TESTER_GROUPS = 'internal-testers'
      }
      steps {
        withCredentials([file(credentialsId: 'firebase-sa', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh '''
            npm install -g firebase-tools || true
            APK=$(ls app/build/outputs/apk/*/*.apk | head -n1)
            echo "Distributing $APK"
            firebase appdistribution:distribute "$APK" \
              --app "$FIREBASE_APP_ID" \
              --groups "$TESTER_GROUPS" \
              --release-notes "Automated build from Jenkins: $BUILD_TAG"
          '''
        }
      }
    }

    stage('MobSF Analysis (APK)') {
      when {
        allOf {
          environment name: 'RUN_MOBSF', value: 'true'
          environment name: 'RUN_BUILD', value: 'true'
        }
      }
      environment {
        APK_PATH = ''
      }
      steps {
        withCredentials([string(credentialsId: 'mobsf-api-key', variable: 'MOBSF_API_KEY')]) {
          sh '''
            set -e
            APK=$(ls app/build/outputs/apk/*/*.apk | head -n1)
            echo "APK: $APK"
            mkdir -p $REPORT_DIR/mobsf

            # Upload APK
            UPLOAD_JSON=$(curl -s -X POST "$MOBSF_URL/api/v1/upload" \
              -H "Authorization: $MOBSF_API_KEY" \
              -F "file=@$APK")
            echo "$UPLOAD_JSON" > $REPORT_DIR/mobsf/upload.json

            # Extract hash (use python for robust JSON parsing)
            HASH=$(python3 -c "import sys, json; print(json.load(sys.stdin).get('hash',''))" <<< "$UPLOAD_JSON")
            if [ -z "$HASH" ]; then
              echo "Failed to get MobSF hash from upload response"
              exit 1
            fi
            echo "MobSF HASH: $HASH"

            # Start scan
            SCAN_JSON=$(curl -s -X POST "$MOBSF_URL/api/v1/scan" \
              -H "Authorization: $MOBSF_API_KEY" \
              -H "Content-Type: application/json" \
              -d "{\"scan_type\": \"apk\", \"hash\": \"$HASH\"}")
            echo "$SCAN_JSON" > $REPORT_DIR/mobsf/scan.json

            # Generate PDF report
            curl -s -X POST "$MOBSF_URL/api/v1/download_pdf" \
              -H "Authorization: $MOBSF_API_KEY" \
              -H "Content-Type: application/json" \
              -d "{\"hash\": \"$HASH\"}" \
              --output $REPORT_DIR/mobsf/MobSF-Report.pdf || true
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "$REPORT_DIR/mobsf/*", fingerprint: true, allowEmptyArchive: true
          publishHTML(target: [
            reportDir: "$REPORT_DIR/mobsf",
            reportFiles: 'upload.json,scan.json',
            reportName: 'MobSF (raw JSON)'
          ])
        }
      }
    }
  } // stages

  post {
    always {
      // Collect Gradle build scans & logs if present
      archiveArtifacts artifacts: "app/build/outputs/**, **/build/reports/**, **/lint-results*.xml", allowEmptyArchive: true
    }
    success {
      echo "✅ Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    failure {
      echo "❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
