pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }

  environment {
    PATH   = "/opt/homebrew/opt/ruby@3.3/bin:/opt/homebrew/lib/ruby/gems/3.3.0/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"

    LANG   = 'en_US.UTF-8'
  }

  stages {


    stage('Tooling sanity') {
      steps {
        sh '''
          set -euo pipefail
          echo "=== Repo root ==="; ls -la
          echo "=== Xcode ==="; xcodebuild -version
          echo "=== Ruby ==="; ruby -v
          echo "PATH=$PATH"
          echo "=== Fastlane ==="; which fastlane; fastlane --version
          echo "=== jq ==="; jq --version
          echo "=== Simulators ==="; xcrun simctl list devices | head -n 30
        '''
      }
    }

stage('Bundle install') {
  steps {
    sh '''
      set -euo pipefail

      if [ -f Gemfile ]; then
        # Read the required bundler version from Gemfile.lock (BUNDLED WITH)
        if [ -f Gemfile.lock ]; then
          BUNDLER_VER=$(awk '/^BUNDLED WITH/{getline; gsub(/^ +| +$/,""); print}' Gemfile.lock)
        else
          BUNDLER_VER=""
        fi

        # Install bundler into a local, writable gem home
        GEM_HOME="${WORKSPACE}/.gems"
        GEM_PATH="${GEM_HOME}"
        export GEM_HOME GEM_PATH
        export PATH="${GEM_HOME}/bin:${PATH}"

        if [ -n "${BUNDLER_VER:-}" ]; then
          echo "[info] Installing bundler ${BUNDLER_VER} into ${GEM_HOME}"
          gem install --no-document -i "${GEM_HOME}" bundler -v "${BUNDLER_VER}"
        else
          echo "[info] Installing latest bundler into ${GEM_HOME}"
          gem install --no-document -i "${GEM_HOME}" bundler
        fi

        # Use vendor/bundle inside the workspace (no system writes)
        bundle config set path 'vendor/bundle'
        bundle install --jobs=4
      fi
    '''
  }
}



    stage('Detect simulator & project') {
      steps {
        sh '''
          set -euo pipefail

          # Pick a simulator
          DEVICE_NAME=$(xcrun simctl list devices -j | jq -r \
            '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | select(.state=="Shutdown") | .name' | head -n1)
          [ -n "${DEVICE_NAME:-}" ] || DEVICE_NAME=$(xcrun simctl list devices -j | jq -r \
            '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | .name' | head -n1)
          [ -n "${DEVICE_NAME:-}" ] || { echo "[error] No available iPhone simulator"; exit 1; }

          UDID=$(xcrun simctl list devices -j | jq -r --arg NAME "$DEVICE_NAME" \
            '.devices[] | .[] | select(.name==$NAME) | .udid' | head -n1)
          [ -n "${UDID:-}" ] || { echo "[error] No UDID for $DEVICE_NAME"; exit 1; }

          echo "$DEVICE_NAME" > .sim_device
          echo "$UDID"       > .sim_udid
          echo "[info] Chosen Simulator: $DEVICE_NAME ($UDID)"

          # Detect first .xcodeproj (or swap to .xcworkspace if you use CocoaPods)
          rm -f .xcodeproj_path .scheme
          XCODEPROJ=$(find . -maxdepth 3 -type d -name "*.xcodeproj" | head -n1 || true)
          [ -n "${XCODEPROJ:-}" ] || { echo "[error] No *.xcodeproj found"; ls -la; exit 1; }
          XCODEPROJ=$(cd "$(dirname "$XCODEPROJ")" && pwd)/"$(basename "$XCODEPROJ")"
          echo "$XCODEPROJ" > .xcodeproj_path

          SCHEME=$(xcodebuild -list -json -project "$XCODEPROJ" | jq -r '.project.schemes[0]')
          [ -n "${SCHEME:-}" ] || { echo "[error] Could not detect scheme"; exit 1; }
          echo "$SCHEME" > .scheme

          echo "[info] Project: $XCODEPROJ | Scheme: $SCHEME"
        '''
      }
    }

    stage('Unit Tests (Fastlane)') {
      steps {
        sh '''
          set -euo pipefail
          SIM_DEVICE=$(cat .sim_device)
          SIM_UDID=$(cat .sim_udid)
          XCODEPROJ=$(cat .xcodeproj_path)
          SCHEME=$(cat .scheme)

          # Boot the simulator (ignore if already booted)
          xcrun simctl boot "$SIM_UDID" || true
          open -a Simulator || true

          SIM_DEVICE="$SIM_DEVICE" \
          SIM_UDID="$SIM_UDID" \
          XCODEPROJ="$XCODEPROJ" \
          SCHEME="$SCHEME" \
          bundle exec fastlane unit_test
        '''
      }
    }

    stage('Build (Debug - Simulator)') {
      steps {
        sh '''
          set -euo pipefail
          SIM_DEVICE=$(cat .sim_device)
          SIM_UDID=$(cat .sim_udid)
          XCODEPROJ=$(cat .xcodeproj_path)
          SCHEME=$(cat .scheme)

          SIM_DEVICE="$SIM_DEVICE" \
          SIM_UDID="$SIM_UDID" \
          XCODEPROJ="$XCODEPROJ" \
          SCHEME="$SCHEME" \
          bundle exec fastlane build_debug_sim
        '''
      }
    }
  }

  post {
    always {
      junit allowEmptyResults: true, testResults: 'fastlane/test_output/report.junit'
      archiveArtifacts artifacts: 'fastlane/test_output/**/*, build/logs/**/*, build/Artifacts/**/*', allowEmptyArchive: true
      echo 'Pipeline finished.'
    }
  }
}
