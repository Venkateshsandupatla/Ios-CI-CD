pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }

  environment {
    // Ensure Jenkins sees Homebrew/CLI tools (Apple Silicon uses /opt/homebrew/bin)
    PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"
    LC_ALL = 'en_US.UTF-8'
    LANG   = 'en_US.UTF-8'
  }

  stages {
    stage('Checkout sample app') {
      steps {
        git url: 'https://github.com/bitrise-io/sample-apps-ios-simple-objc.git'
      }
    }

    stage('Tooling sanity') {
      steps {
        sh '''
          set -euo pipefail
          echo "=== Xcode ==="; xcodebuild -version
          echo "=== Ruby ==="; ruby -v
          echo "PATH=$PATH"
          echo "=== Fastlane ==="; which fastlane; fastlane --version
          echo "=== Cocoapods ==="; pod --version || true
          echo "=== jq ==="; jq --version
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

    // ---------- CHANGED STAGE #1 ----------
   stage('Fastlane setup') {
  steps {
    sh '''
      set -euo pipefail
      mkdir -p fastlane

      # Pick first available iPhone simulator in Shutdown state (stable)
      DEVICE_NAME=$(xcrun simctl list devices -j | \
        jq -r '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | select(.state=="Shutdown") | .name' \
        | head -n1)
      if [ -z "${DEVICE_NAME:-}" ]; then
        echo "[warn] No Shutdown iPhone found; falling back to first available iPhone"
        DEVICE_NAME=$(xcrun simctl list devices -j | \
          jq -r '.devices[] | .[] | select(.name|test("^iPhone")) | select(.isAvailable==true) | .name' \
          | head -n1)
      fi
      [ -n "${DEVICE_NAME:-}" ] || { echo "[error] No available iPhone simulator"; exit 1; }

      UDID=$(xcrun simctl list devices -j | \
        jq -r --arg NAME "$DEVICE_NAME" '.devices[] | .[] | select(.name==$NAME) | .udid' | head -n1)
      [ -n "${UDID:-}" ] || { echo "[error] No UDID for $DEVICE_NAME"; exit 1; }

      echo "$DEVICE_NAME" > .sim_device
      echo "$UDID"       > .sim_udid
      echo "[info] Chosen Simulator: $DEVICE_NAME ($UDID)"

      # --- Detect Xcode project and scheme ---
      XCODEPROJ=$(ls -1 *.xcodeproj | head -n1 || true)
      [ -n "${XCODEPROJ:-}" ] || { echo "[error] No .xcodeproj found in repo root"; ls -la; exit 1; }
      echo "$XCODEPROJ" > .xcodeproj

      SCHEME=$(xcodebuild -list -json -project "$XCODEPROJ" | jq -r '.project.schemes[0]')
      [ -n "${SCHEME:-}" ] || { echo "[error] Could not detect scheme from $XCODEPROJ"; exit 1; }
      echo "$SCHEME" > .scheme

      echo "[info] Project: $XCODEPROJ | Scheme: $SCHEME"

      # --- Write Fastfile (reads SIM_DEVICE/XCODEPROJ/SCHEME from ENV) ---
      rm -f fastlane/Fastfile
cat > fastlane/Fastfile <<'EOF'
default_platform(:ios)

platform :ios do
  desc "Run unit tests on simulator"
  lane :unit_test do
    device  = ENV['SIM_DEVICE']  || "iPhone 16 Pro"
    proj    = ENV['XCODEPROJ']   || Dir['*.xcodeproj'].first
    scheme  = ENV['SCHEME']      || "sample-apps-ios-simple-objc"

    UI.message("Using project: #{proj}, scheme: #{scheme}, device: #{device}")

    scan(
      project: proj,
      scheme: scheme,
      devices: [device],
      clean: true,
      build_for_testing: true,
      output_types: "junit",
      output_directory: "fastlane/test_output"
    )
  end
end
EOF

      echo "[info] Fastfile written:"
      nl -ba fastlane/Fastfile

      # Parse check
      fastlane lanes
    '''
  }
}



    // (Optional) small probe to show what we picked
    stage('Simulator probe') {
      steps {
        sh '''
          set -e
          echo "Chosen device: $(cat .sim_device 2>/dev/null || echo 'N/A')"
          echo "Chosen UDID:   $(cat .sim_udid 2>/dev/null || echo 'N/A')"
          xcrun simctl list devices | head -n 50
        '''
      }
    }

    // ---------- CHANGED STAGE #2 ----------
stage('Unit Tests (Simulator)') {
  steps {
    sh '''
      set -euo pipefail
      SIM_DEVICE=$(cat .sim_device)
      SIM_UDID=$(cat .sim_udid)
      XCODEPROJ=$(cat .xcodeproj)
      SCHEME=$(cat .scheme)

      echo "[info] Using simulator: $SIM_DEVICE ($SIM_UDID)"
      echo "[info] Using project: $XCODEPROJ | scheme: $SCHEME"

      # Boot the simulator if needed
      xcrun simctl boot "$SIM_UDID" || true
      open -a Simulator || true

      # Pass vars as environment to fastlane
      SIM_DEVICE="$SIM_DEVICE" XCODEPROJ="$XCODEPROJ" SCHEME="$SCHEME" fastlane unit_test
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
