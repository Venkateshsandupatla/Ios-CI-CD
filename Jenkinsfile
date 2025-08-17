pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }

  environment {
    // Add Homebrew + common gem/bin dirs to PATH for the jenkins shell
    PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
    LC_ALL = 'en_US.UTF-8'
    LANG   = 'en_US.UTF-8'
  }

  stages {
    stage('Checkout sample app') {
      steps {
        git url: 'https://github.com/Venkateshsandupatla/sample-apps-ios-simple-objc.git'
      }
    }

    stage('Tooling sanity') {
      steps {
        sh '''
          set -euo pipefail
          echo "=== Xcode ==="; xcodebuild -version
          echo "=== Ruby ==="; ruby -v
          set -e
          echo "PATH=$PATH"
          which fastlane || true
          echo "=== Fastlane ==="; fastlane --version
          echo "=== Cocoapods ==="; pod --version || true
          echo "=== Simulators ==="; xcrun simctl list devices | head -n 50
        '''
      }
    }

    stage('Dependencies (Pods if present)') {
      steps {
        sh '''
          set -euo pipefail
          if [ -f "Podfile" ]; then
            pod install
          fi
        '''
      }
    }

    stage('Fastlane setup') {
      steps {
        sh '''
          set -euo pipefail
          mkdir -p fastlane

        # Pick first available iPhone simulator in Shutdown state (stable)
          DEVICE_NAME=$(xcrun simctl list devices -j | \
            jq -r '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | select(.state=="Shutdown")) | .name' \
            | head -n1)

          if [ -z "${DEVICE_NAME:-}" ]; then
            echo "[warn] No Shutdown iPhone found; falling back to first available iPhone"
            DEVICE_NAME=$(xcrun simctl list devices -j | \
              jq -r '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | .name' \
              | head -n1)
          fi

          if [ -z "${DEVICE_NAME:-}" ]; then
            echo "[error] Could not find any available iPhone simulator"; exit 1
          fi

          # Find its UDID
          UDID=$(xcrun simctl list devices -j | \
            jq -r --arg NAME "$DEVICE_NAME" '.devices[] | .[] | select(.name==$NAME) | .udid' \
            | head -n1)

          if [ -z "${UDID:-}" ]; then
            echo "[error] Could not determine UDID for $DEVICE_NAME"; exit 1
          fi

          echo "$DEVICE_NAME" > .sim_device
          echo "$UDID"       > .sim_udid
          echo "[info] Chosen Simulator: $DEVICE_NAME ($UDID)"


          cat > fastlane/Fastfile <<'RUBY'
          default_platform(:ios)
          platform :ios do
            desc "Run unit tests on simulator"
            lane :unit_test do
              # Device is passed via ENV['SIM_DEVICE'] from Jenkins stage
              device = ENV['SIM_DEVICE'] || "iPhone 16 Pro"
              scan(
                scheme: "sample-apps-ios-simple-objc",
                devices: [device],
                build_for_testing: true,
                clean: true,
                output_types: "junit",
                output_directory: "fastlane/test_output"
              )
            end
          end
          RUBY

          # record the chosen device for visibility
          echo "$DEVICE_NAME" > .sim_device
        '''
      }
    }

    stage('Unit Tests (Simulator)') {
      steps {
        sh '''
          set -euo pipefail
          SIM_DEVICE=$(cat .sim_device)
          SIM_UDID=$(cat .sim_udid)
          echo "[info] Using simulator: $SIM_DEVICE ($SIM_UDID)"

          # Boot the simulator if needed (ignore if already booted)
          xcrun simctl boot "$SIM_UDID" || true
          open -a Simulator || true

          # Run tests via Fastlane
          fastlane unit_test SIM_DEVICE="$SIM_DEVICE"

        '''
      }
    }
  }

  post {
    always {
      junit allowEmptyResults: true, testResults: 'fastlane/test_output/report.junit'
      archiveArtifacts artifacts: 'fastlane/test_output/**/*', allowEmptyArchive: true
      echo 'Sanity pipeline finished.'
    }
  }
}
