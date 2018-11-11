# The PIN Project CI/CD gyakorlat

## Előkövetelmény

> Töltsük fel a The PIN Project legutolsó állapotát egy saját ``` GitHub  ``` repositoryba.

Az utolsó állapot elérhető:
https://github.com/AutSoft/AndroidKotlinPart2/tree/master/ThePinProject

> Nyissuk meg a projektet a saját repository-t felhasználva!

```New Project``` -> ```Project from Version Control``` -> ```Git``` -> ```URL```: Saját repository URL

## The PIN Project vizsgálata

> Készítsünk három új ```Run Configuration```-t ahhoz, hogy minden tesztet futtatni tudjunk.

1. JUnit tests ```Add New Configuration``` -> ```Android JUnit``` ->```All in package```
2. Instrumentation tests ```Add New Configuration``` -> ```Android Instrumentation Test``` -> ```All in module```
3. All tests ```Add New Configuration``` -> ```Android Instrumentation Test``` -> ```All in module``` -> ```Before launch``` -> ```Run Unit Tests```

> Futtassuk a tesztjeinket

> Pro Tip: Az IDEA-ban lehetőség van ```Shared Run Configuration```-ök készítésére, ami commitolható a VCS-be, így bárki számára elérhető lesz az adott futtatási konfiguráció

## Package név és alkalmazás azonosító módosítása

> Nevezzük át a az ```AndroidManifest.xml```-ben az alkalmazás package nevét ```hu.autsoft.hwsw.[SAJÁT_NÉV].pin```-re!

> Próbáljuk meg futtatni a tesztjeinket mindkét ```Flavor```-rel, javítsuk meg a hibákat!

1. Importáljuk a helyes ```R.*``` azonosítókat
2. Frissítsük az ```Activity``` neveket az ```AndroidManifest.xml```-ben

## Aláíró kulcsok generálása

> Készítsük el az alkalmazás ```Debug```  kulcsát!

```Generate Sign bundle or APK``` -> ```APK``` -> ```Create New```

Hely: **PROJECT_ROOT/assets**

Név: **debug.keystore**

Alias: **androiddebugkey**

Keypass: **android**

Storepass: **android**

> Készítsük el az alkalmazás ```Release```  kulcsát, ahol már saját adatokat használunk!

> Az app modul ```build.gradle```-be vegyük fel az új aláírás konfigurációkat

```
signingConfigs {
    debug {
        storeFile file("../assets/debug.keystore")
    }
    release {
        storeFile file("../assets/release.keystore")
        storePassword "TPP1234"
        keyAlias "ThePinProject"
        keyPassword "TPP1234"
    }
}

buildTypes {
    debug {
        signingConfig signingConfigs.debug
    }
    release {
        signingConfig signingConfigs.release
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
}
```

> Próbáljuk meg ```Release``` módban futtatni az alkalmazást ```Prod``` ```Flavor```-rel

## CircleCI integráció

### Előkövetelmény

[```CircleCI ```](https://circleci.com/) és a ```GitHub``` account összekapcsolása (SignUp with GitHub).

### Projekt inicializálása

A CircleCI webes felületén végezzük el az alábbiakat:

1. ```Set Up Project``` a korábban létrehozott projektre
2. Operating System -> ```Linux```
3. Language -> ```Gradle (Java)```

### CircleCI konfiguráció leíró létrehozása

> Hozzuk létre a CircleCI konfigurációt leíró fájlt!

Hely: **PROJECT_ROOT/circleci**

Név: **config.yml**

```
version: 2
jobs:
  build:
    working_directory: ~/code
    docker:
      - image: circleci/android:api-28-alpha
    environment:
      JVM_OPTS: -Xmx3200m
    steps:
      - checkout
      - restore_cache:
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run:
         name: Chmod permissions
         command: sudo chmod +x ./gradlew
      - run:
          name: Download Dependencies
          command: ./gradlew androidDependencies
      - save_cache:
          paths:
            - ~/.gradle
          key: jars-{{ checksum "build.gradle" }}-{{ checksum  "app/build.gradle" }}
      - run:
          name: Run Unit Tests
          command: ./gradlew lint test
      - store_artifacts:
          path: app/build/reports
          destination: reports
      - store_test_results:
          path: app/build/test-results
          destination: test-results
      - run:
          name: Build Debug
          command: ./gradlew assembleDebug assembleAndroidTest
      - run:
          name: Build Release
          command: ./gradlew assembleProdRelease
      - store_artifacts:
                path: app/build/outputs/apk
                destination: apks
```


> Kattintsunk a ```Start building```-re, amivel elindul az első buildünk!

> Javítsuk ki a problémát: refaktoráljunk, hogy a ```MockPinValidator``` osztály elérhető legyen a ```main``` könyvtárból is minden konfigurációban!

> Pusholjuk a változtatást, majd gyönyörködjünk, ahogy kizöldül a build.

> Vizsgáljuk meg a ```Job``` által létrehozott könyvtárakat, riportokat.

## Firebase Test Lab integráció

Ahhoz, hogy a ```CircleCI```  jogosultságot szerezzen a ```Firebase Test Lab```-on végzett műveletekhez, regisztrálnunk kell az alkalmazást a [```Firebase console```](https://console.firebase.google.com/) -on.

### Előkövetelmény

Érvényes ```Google``` account összekötve a [```Firebase console```](https://console.firebase.google.com/)-al és [```Google Developer Console```](https://console.developers.google.com/)-al

### UI Tesztek futtatása manuálisan

> Hozzunk létre egy új projektet a ```Firebase Console```-on ```ThePinProject``` néven

> Jegyezzük fel a létrehozott ```Project ID```-t! Pl. **thepinproject-dbd56**

1. ```CircleCI Job``` eredményéből töltsük le a **app-mock-debug.apk** és az **app-mock-debug-androidTest.apk** alkalmazásokat
2. Töltsük fel a ```Firebase Test Lab```-ba és futtassuk manuálisan

### UI Tesztek futtatása automatizáltan

> Válasszuk ki a létrehozott projektet a ```Google Developer Console```-on és engedélyezzük a következő API-kat:

* **Google Cloud Testing API**
* **Cloud Tool Results API**

> Hozzunk létre egy új ```Service Account Key```-t!

Service account name: **circleci-instrumentation-tests**

Role: **Project-Editor**

> Jegyezzük fel az ```Email```-t, illetve vegyük elő a korábban megjegyzett ```Project ID```-t !

Például:

* Project ID: **thepinproject-dbd56**
* Email: **circleci-instrumentation-tests@thepinproject-dbd56.iam.gserviceaccount.com**

> A generált ```JSON``` kulcsot kódoljuk ```BASE64```-el például a következő online eszközzel: https://www.base64encode.net/

> A kiementet adjuk hozzá a projekthez egy új környezeti változóként a projekt beállításoknál: ```Environment Variable``` -> ```GCLOUD_SERVICE_KEY```

> Egészítsük ki a CircleCI configot a következő kódrészlettel a ```Build Debug``` és ```Build Release``` lépések közé

```
# Firebase Test Lab
    - run:
        name: Export Google Cloud Service key environment variable
        command: echo 'export GCLOUD_SERVICE_KEY="$GCLOUD_SERVICE_KEY"' >> $BASH_ENV
    - run:
        name: Decode Google Cloud credentials
        command: echo $GCLOUD_SERVICE_KEY | base64 -di > ${HOME}/client-secret.json
    - run:
        name: Set Google Cloud target project
        command: gcloud config set project [PROJECT_ID]
    - run:
        name: Authenticate with Google Cloud
        command: gcloud auth activate-service-account [GCLOUD_SERVICE_EMAIL] --key-file ${HOME}/client-secret.json
    - run:
        name: Run instrumented test on Firebase Test Lab
        command: gcloud firebase test android run --type instrumentation --app app/build/outputs/apk/mock/debug/app-mock-debug.apk --test app/build/outputs/apk/androidTest/mock/debug/app-mock-debug-androidTest.apk --device model=sailfish,version=28,locale=en_US,orientation=portrait --timeout 8m
```

> Töltsük ki a **PROJECT_ID**-t és a **GCLOUD_SERVICE_EMAIL**-t