# Nexus Repository Manager on DigitalOcean

This repository contains the notes and walkthrough for **Module 6**, where the focus is on installing and configuring **Nexus Repository Manager** on a DigitalOcean droplet, managing firewall access, creating users and roles, and publishing Java artifacts with **Gradle** and **Maven**.

The module builds on the DigitalOcean deployment workflow from the earlier lecture repo and moves into artifact management with Nexus.

## Objectives

By the end of this module, you should be able to:

- Create a DigitalOcean droplet sized for Nexus.
- Configure firewall rules for SSH and Nexus access.
- Install Java 17 and Nexus Repository Manager.
- Run Nexus using a dedicated non-root user.
- Access the Nexus UI and retrieve the initial admin password.
- Create users, roles, and permissions for artifact publishing.
- Upload `.jar` artifacts to Nexus using Gradle.
- Upload `.jar` artifacts to Nexus using Maven.
- Query Nexus repositories and components using the REST API.
- Understand blob storage, components, assets, and cleanup policies.

## Prerequisites

Before starting, make sure you have:

- A DigitalOcean account and access to create droplets.
- A Linux-based droplet or VM.
- SSH access to the server.
- Basic Linux command-line knowledge.
- Java project examples for Gradle and Maven.
- Open firewall access for required ports.

## 1. Create the Droplet

Create a DigitalOcean droplet with **more memory/capacity**, since Nexus requires more resources than the size of the app.

Recommended setup:

- Create a droplet with enough RAM for Nexus.
- Allow **port 22** in the firewall for SSH access.
- Attach the firewall to the droplet.

## 2. Install Java 17

Nexus requires **Java 17**, so install that version before starting the setup.

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

## 3. Download and Extract Nexus

Download Nexus using the archive URL from the Sonatype site. Make sure to pick up the package URL that matches systems specifications. It will be downloaded in the `/opt` directory ,  Then untar the file.

Example flow:

```bash
cd /opt
wget <nexus-download-url>
tar -zxvf <file-name>
```

After extraction, two important folders are created:

- `nexus-<version>`: contains the Nexus runtime, binaries, libraries, deploy files, and application code.
- `sonatype-work`: contains Nexus configuration, logs, uploaded artifacts, plugins, and repository data.

### Why the `sonatype-work` folder matters

This folder stores the persistent Nexus data, including:

- Repository contents
- Logs
- Plugins
- Configuration-generated data
- Uploaded artifacts

It is also useful for **backup**, and the application can be upgraded independently while preserving stored data as we only upgrade the nexus folder.

## 4. Create a Dedicated Nexus User

Nexus should **not** be run as the `root` user, because that would give the application unnecessary root privileges.

Create a dedicated user such as `nexus`, then assign ownership of both Nexus directories.

Example:

```bash
chown -R nexus:nexus /opt/nexus-3.91.1-04
chown -R nexus:nexus /opt/sonatype-work
```

## 5. Configure Nexus to Run as the Nexus User

Edit or create the `nexus.rc` file and set:

```bash
run_as_user="nexus"
```

This ensures the Nexus service starts under the dedicated account instead of root.

## 6. Start Nexus

Switch to the Nexus user and start the application from the `bin` directory.

Example:

```bash
/opt/nexus-3.91.1-04/bin/nexus start
```

To verify that Nexus is running:

```bash
ps aux | grep nexus
netstat -lntp
```

If `netstat` is not available, install `net-tools` first:

```bash
sudo apt update && sudo apt install net-tools
```

The notes show Nexus running on **port 8081**.

## 7. Open Firewall Access for Nexus

Since Nexus listens on port `8081`, add that port to the droplet firewall so the repository manager can be accessed publicly.

Then open the Nexus UI in the browser using:

```text
http://<droplet-ip>:8081
```

## 8. Initial Login

Login with:

- Username: `admin`
- Password file:

```text
/opt/sonatype-work/nexus3/admin.password
```

Use `cat` or `vim` to read the password file, then sign in through the browser. After the first login, Nexus prompts you to change the password and configure access options.

## 9. Repository Types in Nexus

The notes identify three main repository types:

- **Proxy**
- **Group**
- **Hosted**

For this module, the artifact upload examples use the **Maven hosted repository** already created by default in Nexus.

## 10. Create Users and Roles

Instead of giving developers the admin credentials, create dedicated users and assign only the permissions they need.

Lecture flow:

1. Go to **Settings** and create a new user.
2. Go to **Security > Roles** and create a role.
3. Assign permissions for the target repositories.
4. Attach the role to the user.
5. Use that user’s credentials in Gradle or Maven.

This follows a better security practice than sharing the admin account.

## 11. Publish a JAR with Gradle

Gradle and Maven both publish Java artifacts in **Maven format**, so Nexus can store them in Maven repositories.

### Add publishing support

In `build.gradle`, enable publishing with the `maven-publish` plugin:

```groovy
apply plugin: 'maven-publish'

publishing {
    publications {
        create("maven", MavenPublication) {
            artifact("build/libs/my-app-$version" + ".jar") {
                extension 'jar'
            }
        }
    }

    repositories {
        maven {
            url = uri("http://your-repository-url")
            allowInsecureProtocol = true
            credentials {
                username project.repoUser
                password project.repoPassword
            }
        }
    }
}
```

### Store credentials safely

Do not hardcode credentials in `build.gradle`. Put them in `gradle.properties` instead:

```properties
repoUser=username
repoPassword=xxxxx
```

### Configure the project name and version

In `settings.gradle`, define the project name, such as:

```groovy
rootProject.name = 'my-app'
```

In `build.gradle`, define the version:

```groovy
version = '1.0-SNAPSHOT'
```

### Build and publish

Build the JAR:

```bash
gradle build
```

Then publish it:

```bash
gradle publish
```

The `publish` task is available because of the `maven-publish` plugin configuration.

After publishing, log in to Nexus and browse the Maven snapshot repository to confirm the uploaded `.jar` and generated metadata files.

## 12. Publish a JAR with Maven

For Maven projects, configure deployment in `pom.xml`.

### Add the deploy plugin

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-deploy-plugin</artifactId>
    <version>3.1.2</version>
</plugin>
```

### Configure the snapshot repository

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://<nexus-ip>:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

### Configure Maven credentials

Create `settings.xml` inside the `.m2` directory:

```xml
<settings>
  <servers>
    <server>
      <id>nexus-snapshots</id>
      <username>username</username>
      <password>xxxxx</password>
    </server>
  </servers>
</settings>
```

The repository `id` in `settings.xml` must match the `id` in `pom.xml`.

### Build and deploy

Package the project:

```bash
mvn package
```

Deploy the artifact:

```bash
mvn deploy
```

After deployment, check the Nexus browse view and confirm the artifact appears inside the snapshot repository.

## 13. Nexus REST API

Nexus provides REST endpoints that can be used in automation and CI/CD pipelines to inspect repositories, components, and versions.

Example command to list repositories:

```bash
curl -u user:password -X GET 'http://<nexus-ip>:8081/service/rest/v1/repositories'
```

Example command to fetch a specific component:

```bash
curl -u user:password -X GET 'http://<nexus-ip>:8081/service/rest/v1/components/<component-id>'
```

These endpoints are useful when automating builds, artifact lookups, and deployment pipelines.

## 14. Blob Storage, Components, and Assets

### Blob storage

Nexus uses **blob stores** to manage binary artifact storage.

Blob stores can use:

- Local file storage
- Cloud storage such as AWS S3

The default blob store is file-based storage on the server.

### Components vs assets

The notes distinguish these two concepts:

- **Component**: the uploaded package or logical repository item.
- **Asset**: the individual files that belong to that component.

A component can contain one or more assets.

## 15. Cleanup Policies

Nexus cleanup policies allow old or unused artifacts to be removed from repositories in order to free storage space.

These policies help manage repository growth and are especially useful when working with snapshots and older build outputs.

## Key Commands

```bash
# Extract Nexus archive
tar -zxvf <file-name>

# Change ownership
chown -R nexus:nexus /opt/nexus-3.91.1-04
chown -R nexus:nexus /opt/sonatype-work

# Start Nexus
/opt/nexus-3.91.1-04/bin/nexus start

# Check running process
ps aux | grep nexus

# Check listening ports
netstat -lntp

# Install netstat tools if missing
sudo apt update && sudo apt install net-tools

# Gradle build and publish
gradle build
gradle publish

# Maven package and deploy
mvn package
mvn deploy

# Query Nexus repositories
curl -u user:password -X GET 'http://<nexus-ip>:8081/service/rest/v1/repositories'
```

## Learning Outcomes

After completing this module, you should understand how to:

- Host Nexus on a DigitalOcean droplet.
- Secure it with firewall rules and non-root execution.
- Manage repository users and permissions.
- Publish Java artifacts from Gradle and Maven.
- Use the Nexus API for automation.
- Understand storage concepts such as blob stores, components, and assets.

## Notes

- Java 17 is required for the Nexus version used in this lecture.
- Port `8081` must be opened to access the Nexus web interface.
- Port `22` must remain open for SSH.
- `sonatype-work` contains the important persistent data and should be considered for backup.
- Avoid storing usernames and passwords directly in project files.

## Source

This README was created from the user’s Module 6 lecture notes and follows the style of a practical lab repository README. The Nexus installation flow aligns with DigitalOcean app deployment concepts in which code or services are deployed from a server environment and exposed through configured ports and app access rules [1][2].
