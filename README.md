# CircleCI Android machine image preview docs

We are excited to announce the preview of an Android machine image on CircleCI. The purpose of this repository is to walk you through the setup steps required to use the Android machine image.

## What is the Android machine image on CircleCI?

CircleCI offers multiple kinds of environments for you to run jobs in. The
Android machine image can be used via the [Linux `machine` executor](https://circleci.com/docs/2.0/configuration-reference/#machine-executor-linux), like other
Linux machine images on CircleCI. The Android machine image supports nested virtualization and x86 Android emulators, so it can be used for Android UI testing. It also comes with the Android SDK pre-installed.

## Pre-installed Software

### Android SDK
- sdkmanager
- Android platform 27, 28, 29, 30
- Build tools 30.0.3
- emulator, platform-tools, tools
- NDK (Side-by-side) 21.4.7075529
- cmake 3.6.4111459
- extras;android;m2repository, extras;google;m2repository, extras;google;google_play_service

### Others
- gcloud
- OpenJDK 8, OpenJDK 11 (default)
- maven 3.6.3, gradle 6.8, ant
- nodejs 12.20.1, 14.15.4 (default), 15.6.0
- python 2.7.18, python 3.9.1
- ruby 2.7.1, ruby 3.0.0
- docker 20.10.2, docker-compose 1.28.2
- jq 1.6

## Using the Android machine image

To use the Android machine image, edit your `.circleci/config.yml` file.
Here are example configs, some of which leverage the circleci/android [orb](https://circleci.com/developer/orbs/orb/circleci/android) for a simplified config. More examples
are available [here](https://circleci.com/developer/orbs/orb/circleci/android#usage-examples).

### Simple job example

```yaml
# .circleci/config.yaml
version: 2.1
orbs:
  android: circleci/android@1.0
workflows:
  test:
    jobs:
      # This job uses the Android machine image by default
      - android/run-ui-tests:
          # Use pre-steps and post-steps if necessary
          # to execute custom steps before and afer any of the built-in steps
          system-image: system-images;android-29;default;x86
```

### Example of using the Android machine image in your own job

This example shows how you could modify your existing job to run UI tests.

```yaml
# .circleci/config.yml
version: 2.1
orbs:
  android: circleci/android@1.0
jobs:
  test:
    # Use the Android machine image
    executor:
      name: android/android-machine
      resource-class: large
    steps:
      - checkout
      # Creates an AVD and starts up the emulator using the AVD.
      # While the emulator is starting up, the gradle cache will
      # be restored and the Android app will be assembled.
      # When the emulator is ready, UI tests will be run.
      # After the tests are run, the gradle cache will be saved (if it
      # hasn't been saved before)
      - android/start-emulator-and-run-tests:
          system-image: system-images;android-29;default;x86
      #   The cache prefix can be overridden
      #   restore-gradle-cache-prefix: v1a
      #
      #   The command to be run, while waiting for emulator startup, can be overridden
      #   post-emulator-launch-assemble-command: ./gradlew assembleDebugAndroidTest
      #
      #   The test command can be overridden
      #   test-command: ./gradlew connectedDebugAndroidTest
workflows:
  test:
    jobs:
      - test
```

### Example of using more granular orb commands

This example shows how you can use more granular orb commands to
achieve what the [start-emulator-and-run-tests](https://circleci.com/developer/orbs/orb/circleci/android#commands-start-emulator-and-run-tests) command does.

```yaml
# .circleci/config.yml
version: 2.1
orbs:
  android: circleci/android@1.0
jobs:
  test:
    executor:
      name: android/android-machine
      resource-class: large
    steps:
      - checkout
      # Create an AVD named "myavd"
      - android/create-avd:
          avd-name: myavd
          system-image: system-images;android-29;default;x86
          install: true
      # By default, after starting up the emulator, a cache will be restored,
      # "./gradlew assembleDebugAndroidTest" will be run and then a script
      # will be run to wait for the emulator to start up.
      # Specify the "post-emulator-launch-assemble-command" command to override
      # the gradle command run, or set "wait-for-emulator" to false to disable
      # waiting for the emulator altogether.
      - android/start-emulator:
          avd-name: myavd
          no-window: true
          restore-gradle-cache-prefix: v1a
      # Runs "./gradlew connectedDebugAndroidTest" by default.
      # Specify the "test-command" parameter to customize the command run.
      - android/run-tests
      - android/save-gradle-cache:
          cache-prefix: v1a
workflows:
  test:
    jobs:
      - test
```

### No-orb example

Here is an example of using the image, without using the circleci/android [orb](https://circleci.com/developer/orbs/orb/circleci/android). These steps are similar
to what is run when you use the [run-ui-tests](https://circleci.com/developer/orbs/orb/circleci/android#jobs-run-ui-tests) job of the orb.

```yaml
# .circleci/config.yml
version: 2.1
jobs:
  build:
    machine:
      image: android:202102-01
    # To optimize build times, we recommend "large" and above for Android-related jobs
    resource_class: large
    steps:
      - checkout
      - run:
          name: Create avd
          command: |
            SYSTEM_IMAGES="system-images;android-29;default;x86"
            sdkmanager "$SYSTEM_IMAGES"
            echo "no" | avdmanager --verbose create avd -n test -k "$SYSTEM_IMAGES"
      - run:
          name: Launch emulator
          command: |
            emulator -avd test -delay-adb -verbose -no-window -gpu swiftshader_indirect -no-snapshot -noaudio -no-boot-anim
          background: true
      - run:
          name: Generate cache key
          command: |
            find . -name 'build.gradle' | sort | xargs cat |
            shasum | awk '{print $1}' > /tmp/gradle_cache_seed
      - restore_cache:
          key: gradle-v1-{{ arch }}-{{ checksum "/tmp/gradle_cache_seed" }}
      - run:
          # run in parallel with the emulator starting up, to optimize build time
          name: Run assembleDebugAndroidTest task
          command: |
            ./gradlew assembleDebugAndroidTest
      - run:
          name: Wait for emulator to start
          command: |
            circle-android wait-for-boot
      - run:
          name: Disable emulator animations
          command: |
            adb shell settings put global window_animation_scale 0.0
            adb shell settings put global transition_animation_scale 0.0
            adb shell settings put global animator_duration_scale 0.0
      - run:
          name: Run UI tests (with retry)
          command: |
            MAX_TRIES=2
            run_with_retry() {
               n=1
               until [ $n -gt $MAX_TRIES ]
               do
                  echo "Starting test attempt $n"
                  ./gradlew connectedDebugAndroidTest && break
                  n=$[$n+1]
                  sleep 5
               done
               if [ $n -gt $MAX_TRIES ]; then
                 echo "Max tries reached ($MAX_TRIES)"
                 exit 1
               fi
            }
            run_with_retry 
      - save_cache:
          key: gradle-v1-{{ arch }}-{{ checksum "/tmp/gradle_cache_seed" }}
          paths:
            - ~/.gradle/caches
            - ~/.gradle/wrapper
workflows:
  build:
    jobs:
      - build
```

## Limitations

* There may be up to 2 mins of spin-up time before your job actually starts running. This time will decrease as more preview customers start using the
Android image.
* Only one image is currently available, `android:202102-01`. It contains most of the tools you’ll likely need for Android development. If there is software you require that’s not available in the image, please [open an issue](https://github.com/CircleCI-Public/android-image-preview-docs/issues) to let us know.
* We may change and update the pre-installed software on the `android:202102-01` image without prior notice during the preview period. Once the preview period is over, the images for Android images will be stable and will follow our standard image release cadence.

## Pricing

For pricing information, refer to the Linux machine executors under the “Linux VM" section on the [pricing page](https://circleci.com/pricing/).

## How to provide feedback

Please [open an issue](https://github.com/CircleCI-Public/android-image-preview-docs/issues) with any feedback. The specific areas where we would appreciate feedback:

* Software pre-installed in the image. Do you find everything you need in the image?
* Are there any bugs that you’ve noticed in the pre-installed software?
