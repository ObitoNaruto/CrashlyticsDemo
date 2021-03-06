Fabric Crashlytics Kit Setup Demo [![Build Status](https://travis-ci.org/plastiv/CrashlyticsDemo.svg)](https://travis-ci.org/plastiv/CrashlyticsDemo)
===========

Project shows how to:

* setup Fabric Crashlytics for Android
* disable Crashlytics for the debug builds
* hide `apiSecret` and `apiKey` from source control

About Crashlytics
------

> The most powerful, yet lightest weight crash reporting solution.

Crashlytics is one of the crash reporting tools available to collect crash info from the user devices. It is likely that you are using one of the alternatives already: Acra, Bugsense, Crittercism, etc. Crashlytics have all the core features for crash reporting tool: Proguard deobfuscation, support for both handled and unhandled exceptions and no quota limits with free account.

Setup Crashlytics
-----

First of all we need to login at Fabric dashboard [get.fabric.io](https://get.fabric.io/) and download Android Studio plugin. Successful authorization within Android Studio plugin is required to continue. Plugin will provide the credentials which will be used later at CI and plugin can be deleted after the very first account signup.

We enable Crashlytics by adding fabric-tools plugin and crashlytics-sdk compile dependency to our project build script. Make sure to add `maven.fabric.io` repository because Fabrics sources are private and its binaries are not available at maven central.

_app/build.gradle_
```gradle
buildscript {
    repositories {
        maven { url 'https://maven.fabric.io/public' }
    }
    dependencies {
        // to see latest version available:
        // https://maven.fabric.io/public/io/fabric/tools/gradle/maven-metadata.xml
        classpath 'io.fabric.tools:gradle:1.21.6'
    }
}
apply plugin: 'io.fabric'

repositories {
    maven { url 'https://maven.fabric.io/public' }
}
dependencies {
    // to see latest version available:
    // https://maven.fabric.io/public/com/crashlytics/sdk/android/crashlytics/maven-metadata.xml
    compile('com.crashlytics.sdk.android:crashlytics:2.5.7@aar') {
      transitive = true
    }
}
```

Start Crashlytics kit from application entry point and that's all is needed to track unhandled exceptions at Crashlytics dashboard.

_src/java/com.github.plastiv.crashlyticsdemo.CrashlyticsDemoApplication_
```java
public class CrashlyticsDemoApplication extends Application {

  @Override
  public void onCreate() {
    super.onCreate();
    Fabric.with(this, new Crashlytics());
  }
}
```

Wait a second, if that's all what needed, how does Crashlytics authorize SDK and knows where to send those crashes? Glad you asked.

Fabric gradle plugin generates additional files with supporting information which are used at the runtime then to get `apiKey` from code. After building the project we can see next autogenerated modification to sources:

_app/build/intermediates/assets/release/crashlytics-build.properties_
```ini
version_name=0.0.1
package_name=com.github.plastiv.crashlyticsdemo
build_id=d8ba607c-1abc-49c5-9f55-2f45b7de083b
version_code=1
app_name=Crashlytics Demo
```

_app/build/intermediates/res/merged/release/values/com_crashlytics_build_id.xml_
```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?>
<resources>
<!--
  This file is automatically generated by Crashlytics to uniquely
  identify individual builds of your Android application.

  Do NOT modify, delete, or commit to source control!
-->
<string xmlns:ns0="http://schemas.android.com/tools" name="com.crashlytics.android.build_id" ns0:ignore="UnusedResources,TypographyDashes" translatable="false">ee82bd64-bc3f-468b-9df4-22e64fa79ab4</string>
</resources>
```

_app/build/intermediates/manifests/full/release/AndroidManifest.xml_
```xml
<meta-data
      android:name="com.crashlytics.ApiKey"
      android:value="cc238b2a4866ceb0b008839cdb49a8b77"/>
```

Version name, code and package will be loaded from gradle build script. Crashlytics credentials can be controlled from `fabric.properties` file.

_app/fabric.properties_
```ini
apiSecret=7c9df6d057e7bb62b17ab364e8115a75fcf7430873b6274bb094d1a8adb
apiKey=cc238b2a4866ceb061b008839cdb49a8b77
```

You can get `apiSecret` and `apiKey` from the fabric dashboard [fabric.io/settings/organizations](https://fabric.io/settings/organizations) where they were created after successful first-time signup with Crashlytics plugin GUI for Android Studio.

Disabling Crashlytics for the debug builds
---

You may want to disable Crashlytics crash logging during debugging on local machine. Because all crashes are directly accessible from adb logcat with no need to use server infrastructure and mail notifications. Disabling could be done with gradle build types or flavors and `ext.enableCrashlytics` parameter.

_app/build.gradle_
```gradle
android {
    buildTypes {
        debug {
            // disable crashlytics
            buildConfigField "boolean", "USE_CRASHLYTICS", "false"
            ext.enableCrashlytics = false
        }
        release {
            // enable crashlytics
            buildConfigField "boolean", "USE_CRASHLYTICS", "true"
            ext.enableCrashlytics = true
        }
    }
}
```

After disabling Crashlytics gradle plugin with `ext.enableCrashlytics=false` it stops to create `crashlytics-build.properties` and `com_crashlytics_build_id.xml` files and starting Crashlytics from code results in exception due to missing information. We wrap Fabric factory with declared build config field to prevent it's initialization.

```java
@Override
public void onCreate() {
    if (BuildConfig.USE_CRASHLYTICS) {
        Fabric.with(this, new Crashlytics());
    }
}
```

Providing apiKey and apiSecret on CI
------

We may want to keep your `apiKey` and `apiSecret` private without sharing it within git repository. But we still need this values to be able to run build from CI or other team members machines. We can use the same technique here as when we hiding signingConfig keys from a repository by providing them with project properties.

_app/build.gradle_
```gradle
afterEvaluate {
    initFabricPropertiesIfNeeded()
}

def initFabricPropertiesIfNeeded() {
    def propertiesFile = file('fabric.properties')
    if (!propertiesFile.exists()) {
        def commentMessage = "This is autogenerated fabric property from system environment to prevent key to be committed to source control."
        ant.propertyfile(file: "fabric.properties", comment: commentMessage) {
            entry(key: "apiSecret", value: crashlyticsdemoApisecret)
            entry(key: "apiKey", value: crashlyticsdemoApikey)
        }
    }
}
```

Additional task is added which creates `fabric.properties` file with `crashlyticsdemoApisecret` and `crashlyticsdemoApikey` project properties. With gradle it is really convenient to set project properties:

* Use the -P command-line argument to pass a property to the build script.

`gradlew assemble -PcrashlyticsdemoApikey=cc238b2a4866c96030`

* Or define a `gradle.properties` file and set the property in this file. We can place the file in our project directory or in the `<USER_HOME>/.gradle directory`. The properties defined in the property file in our home directory take precedence over the properties defined in the file in our project directory.

_gradle.properties_ or _~/.gradle/gradle.properties_

```ini
crashlyticsdemoApisecret=0cf7c9df6d057e7bb62b1427ab364e8115a75fcf7430873b6274bb094d1a8adb
crashlyticsdemoApikey=cc238b2a4866c96030
```

* Or use an environment variable of which the name starts with` ORG_GRADLE_PROJECT_` followed by the property name.

`export ORG_GRADLE_PROJECT_crashlyticsdemoApikey=cc238b2a4866c96030`

* Or use the Java system property that starts with `org.gradle.project.` followed by the property name.

`java -Dorg.gradle.project.crashlyticsdemoApikey="cc238b2a4866c96030"`

Because `fabric.properties` is autogenerated now it should be excluded from git tracking.

_.gitignore_
```ini
# Fabric plugin
fabric.properties
```

Custom source sets
-------

We can override Crashlytics source sets when default source folder structure is overridden with gradle so Crashlytics gradle plugin knows where to place autogenerated files. It is useful for projects compatible with Eclipse.

_app/build.gradle_
```gradle
crashlytics.manifestPath = "relative/path/to/manifest"
crashlytics.resPath = "relative/path/to/res"
crashlytics.assetsPath = "relative/path/to/assets"
```

Run demo app
-------

Check yourself, that project doesn't have any of `fabric.properties`, `crashlytics-build.properties`, `com_crashlytics_build_id.xml`
or meta tag at `AndroidManifest.xml` but still both `debug` and `release` apks compiles fine. Because of `apiKey` and `apiSecret`
are provided with environment variables.

[![Environment variables](/images/travis-ci-environment-variables.png)](https://travis-ci.org/plastiv/CrashlyticsDemo)
