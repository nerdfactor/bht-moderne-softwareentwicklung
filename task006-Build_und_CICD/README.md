# Task006: Build und CI/CD

Ich nutze für diese Aufgabe den Code aus der nächsten Aufgabe zu Microservices. Daher wird für dieses gemeinsame Projekt ein neues [Repository auf Github](https://github.com/nerdfactor/bht-bowling-service) erstellt

## Build Tools

### Gradle als neues Build Tool
Ich werde den Microservice in der nächsten Aufgabe mit Spring Boot entwickeln. Bisher habe ich für meine Java Projekte immer Maven als Build Tool verwendet. Vor dem Umstieg auf Gradle habe ich mich immer gescheut. Es schien mir zu viel Aufwand, ohne direkt einen erkennbaren Nutzen zu sehen. Da kommt diese Aufgabe sehr günstig, das jetzt doch mal anzugehen.

Natürlich wäre es jetzt einfach sich von [spring initialir](https://start.spring.io) ein fertiges Gradle Projekt erstellen zu lassen, aber es wäre langweilig. Es reicht mir für die Übung von meinem IDE ein leeres Gradle Projekt zu erzeugen.

![new-gradle-project](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/new-gradle-project.png?raw=true)


Das so erzeugte Projekt ist mir natürlich komplett unbekannt aber Spring bietet eine [hilfreiche Anleitung](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/) zum Einrichten von Spring Boot mit Gradle. Daraus ergibt sich erstmal [ein einfaches Build Skript](https://github.com/nerdfactor/bht-bowling-service/commit/bf3a39c7807027db058f73c64d769132d4658a5f), mit dem zumindest ein Server startet.

```gradle
plugins {
    id("java")
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

group = "eu.nerdfactor"
version = "1.0.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter")

    testImplementation(platform("org.junit:junit-bom:5.9.1"))
    testImplementation("org.junit.jupiter:junit-jupiter")
}

tasks.test {
    useJUnitPlatform()
}
```
Das einzige Problem war dabei nur die Dependencies, die ich aus meinem Maven Projekten ja schon kenne, in der neuen Form noch einmal heraus zu suchen. Als netten Nebeneffekt habe ich dabei auch für alle Artefakte mal die aktuelle Version nehmen können.

```gradle
plugins {
	id("java")
	id("org.springframework.boot") version "3.2.0"
	id("io.spring.dependency-management") version "1.1.4"
	id("io.freefair.lombok") version "8.4"
}

group = "eu.nerdfactor"
version = "1.0.0"

repositories {
	mavenCentral()
}

dependencies {
	implementation("org.springframework.boot:spring-boot-starter-web")

	// database
	implementation("org.springframework.boot:spring-boot-starter-data-jpa")
	runtimeOnly("com.h2database:h2")

	testImplementation(platform("org.junit:junit-bom:5.9.1"))
	testImplementation("org.junit.jupiter:junit-jupiter")
	testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.test {
	useJUnitPlatform()
}
```
Interessant finde ich es, dass es für Gradle sehr viele Plugins der Bibliotheken gibt, die bei der Nutzung unterstützen. Das erscheint mir schon mal ein großer Vorteil gegenüber Maven.

Auf der anderen Seite musste ich aber feststellen, dass die Builds mit Gradle sich sehr viel länger anfühlen. Und auch die Integration der Unit Tests in IntelliJ sind nicht so schön mit Gradle. Ich kann nicht mehr wie gewohnt die Ergebnisse ansehen. Das ist zwar kein harter Hindernisgrund für Gradle, macht einen möglichen Umstieg aber unangenehmer. Was der Entwickler nicht kennt, nutzt er nicht :grin:.


### Sonarqube bei jedem Build
Damit ich auch etwas Komplexeres in Gradle mache, versuche ich es Sonarqube (ja, ich liebe es, seitdem ich es in der Metriken Aufgabe kennengelernt habe) direkt an einen Gradle Build anschließen zu lassen. Eventuell gibt es da auch bereits ein Plugin für, aber ich will das lieber mal selber ausprobieren.

Die Verkettung der beiden Tasks war nach einer kurzen Google Suche nicht so kompliziert.

```gradle
tasks {
	named("build").get().dependsOn("sonar")
}
```
Spannender war es dabei, das Token für den Zugriff auf Sonarqube so in den Buildprozess zu bekommen, dass es nicht als Teil des Build Skripts im Versionskontrollsystem landet.

Mein erster Versuch war es also, das Token in eine extra Datei zu schreiben und mit Hilfe einer Funktion dort auslesen und an der richtigen Stelle einzufügen. Dabei konnte ich auch gleich probieren, wie man direkt im Build Skript komplette Funktionen schreiben kann.

```gradle
sonar {
	properties {
		property("sonar.host.url", "https://sonar.nrdfctr.app")
		property("sonar.projectKey", "bowling-service")
		property("sonar.login", getExternalVar("secrets.gradle", "sonarqubeToken"))
	}
}

fun getExternalVar(fileName : String, varName : String): String {
	val file = rootProject.file(fileName)

	if (file.exists()) {
		val properties = Properties()
		properties.load(FileInputStream(file))
		return properties.getProperty(varName)
	} else {
		throw FileNotFoundException()
	}
}

```
Das funktioniert, aber Gradle hat eine deutlich schönere Möglichkeit einfach die Variablen aus einer anderen Datei zu übernehmen.
```gradle
apply {
	from("secrets.gradle")
}
val sonarqubeToken: String by project
sonar {
	properties {
		property("sonar.host.url", "https://sonar.nrdfctr.app")
		property("sonar.projectKey", "bowling-service")
		property("sonar.login", sonarqubeToken)
	}
}
```
Die Lösung gefällt mir ziemlich gut und erzeugt bei jedem neuen Build eine erfolgreichen Push zu Sonarqube.

![sonar-gradle](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/sonar-gradle.png?raw=true)

## Github Actions
Danach habe ich mir angesehen, was ich so mit Github Actions machen kann. Da CI/CD und komplexere Automatisierung für mich komplett neu ist, wirkt es erstmal sehr erschlagend.

### Sonarqube bei jedem Commit
Nach dem Verketten der Sonarqube und Build Tasks im Gradle Skript, wäre es noch viel besser, wenn bei jedem Commit einfach automatisch Sonarqube gestartet wird. Dann müsste ich nicht mal mehr lokal den Build ausführen.

Die Einrichtung eines Workflows auf Github ist erstmal sehr intuitiv. Danach stochere ich aber mehr im Dunkeln, als dass ich weiß was ich machen muss. Zum Glück gibt es von Sonarqube direkt eine fertige Action, um die Daten zu übertragen.

```yaml
on:
  # Trigger analysis when pushing to your main branches, and when creating a pull request.
  push:
    branches:
      - main
      - master
      - develop
      - 'releases/**'
  pull_request:
      types: [opened, synchronize, reopened]

name: Sonarqube Workflow
jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        # Disabling shallow clones is recommended for improving the relevancy of reporting
        fetch-depth: 0
    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
```
Damit hangele ich mich Schritt für Schritt von einer Fehlermeldung zur nächsten und komme so einem Verständnis für diese Workflows immer näher. Die Erkentnis, dass es sich bei den Workflows im Grund um einen Container handelt, in dem nacheinander Schritte abgearbeitet werden, hat etwas gedauert. Danach war aber alles deutlich klarer.

Neu für mich war auch wie ich Secrets in Github einrichten kann, die dann auch in den Workflows verwendet werden. 

Am Ende habe ich einen Workflow, der erst den Code auscheckt, das Projekt baut und dann die Informationen an Sonarqube überträgt. Und alles direkt bei einem Commit, ohne das ich noch mehr machen muss.

```yaml
on:
  push:
    branches:
      - main

name: Sonarqube Workflow
jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        distribution: 'zulu'
        java-version: '17'
    - name: Gradle build
      run: ./gradlew build
    - name: SonarQube Scan
      uses: sonarsource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_URL }}
```

![sonar-workflow](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/sonar-workflow.png?raw=true)

### Neues Docker Image erstellen
Nachdem ich einen ersten Eindruck der Github Actions bekommen habe, ist der nächste Versuch von Github ein Docker Image bauen zu lassen, damit ich die Anwendung für die nächste Aufgabe zum Testen deployen kann. Auch weil ich erstmal herausfinden muss, wie man eine Spring Boot Anwendung in einen Docker Container bekommt.

Auch hier ist der Umstieg auf Gradle direkt hilfreich. Denn das Spring Boot Gradle Plugin bietet nicht nur die Möglichkeit ein Jar zu bauen, wie es das in Maven auch kann, sondern auch gleich ein Docker Image zu erzeugen. Für den ersten Schritt kann ich also die Frage wie man das Image selber baut zur Seite schieben und es lokal erstellen und in die Registry auf Github übertragen lassen.


![docker-boot-local](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-boot-local.png?raw=true)

Das Image mit Gradle zu bauen war einfach und die Dokumentation von Github erklärt auch gut wie man das Image dorthin pusht.

![docker-boot-push](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-boot-push.png?raw=true)

Als ich das Docker Image in der Registry hatte, war ich schon ziemlich stolz auf mich.

![docker-boot-registry](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-boot-registry.png?raw=true)


Nachdem ich wusste, wie ich das Image manuell in die Registry bekomme, lief es darauf hinaus, im Workflow einige Steps auszutauschen. Und es funktionierte damit auch sehr gut.

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:

name: Docker Workflow

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK
      uses: actions/setup-java@v4
      with:
        distribution: 'zulu'
        java-version: '17'
    - name: Gradle build Docker image
      run: ./gradlew bootBuildImage
    - name: Login to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.REGISTRY_TOKEN }}
    - name: Push Docker image
      run: docker push ghcr.io/nerdfactor/bowling-service:latest
```

![docker-boot-workflow](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-boot-workflow.png?raw=true)

### Docker Workflow für mehreren Architekturen
Mit dem so erstellen Docker Image aus dem Workflow wollte ich es eigentlich belassen. Für die Microservice Aufgabe wollte ich das Image allerdings auf einem meiner Server laufen lassen, damit ich da auch ein schönes greifbares Ergebnis habe. Leider habe ich nicht damit gerechnet, dass mir hier die Server auf ARM Basis einen Strich durch die Rechnung machen.

Das vom Gradle Plugin erstelle Docker Image ist nur für amd64 gebaut und nach der Beschreibung des Plugins gibt es wohl auch keinen Support für ARM. Das liegt wohl irgendwie an dem genutzten Builder Image, aber genau verstehe ich den Grund auch nicht. Anstatt daran weiter herum zu probieren, wollte ich es dann doch einfach direkt selber bauen. Kann ja auch nicht so schwer sein.

Stellt sich heraus: wenn man von Docker noch keine Ahnung hat, dauert es auch lange bis man versteht wie ein einfaches Image erstellt wird. Aber viel Recherche später, habe ich ein Dockerfile, indem das erstelle Jar File ausgeführt wird.

```docker
FROM eclipse-temurin:17-jammy
COPY build/libs/bowling-service.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```
Ich sehe aber schon das Potential mich mit Docker noch sehr viel umfangreicher beschäftigen zu können.

Mit dem neu erstellten Dockerfile habe ich dann erstmal wieder lokal die Images erstellt und versucht in die Registry zu pushen. Das war auch nicht sehr umfangreich, nachdem ich buildx kennengelernt hatte, mit dem das Erstellen für verschiedene Plattformen wohl deutlich einfacher sein soll.

![docker-arch-local](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-arch-local.png?raw=true)

Das Zusammenstellen des Workflows war danach super einfach. Wie die einzelnen Schritte aufeinander aufbauen, hatte ich bei der Übung mit Sonarqube ja schon lernen können. Einen grundlegenden Unterschied zu dem Sonarqube Workflow konnte ich auch noch einbauen. Denn ein neues Docker Image wollte ich nicht bei jedem einzelnen Commit erstellen lassen, sondern habe den Workflow an neue Tags gekoppelt.

```yaml
on:
  push:
    tags:
      - '*'
  workflow_dispatch:

name: Docker Workflow

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'
      - name: Build Jar File
        run: ./gradlew bootJar
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.REGISTRY_TOKEN }}
      - name: Prepare Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Build and Push Docker Image
        run: |
          docker buildx create --use
          docker buildx inspect
          docker buildx build --platform linux/amd64,linux/arm64 --push -t ghcr.io/nerdfactor/bowling-service:latest .
        env:
          DOCKER_CLI_ARCH: amd64
```
![docker-arch-registry](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-arch-registry.png?raw=true)

Das Image gibt es dank dem neuen verbesserten Workflow jetzt auch direkt für adm64 und arm64 und kann auf meinen Testservern genutzt werden.

![docker-portainer](https://github.com/nerdfactor/bht-moderne-softwareentwicklung/blob/main/task006-Build_und_CICD/docker-portainer.png?raw=true)

Was die Anwendung unter https://bowling.nrdfctr.app genau macht, ist aber Teil der Microservice Aufgabe.