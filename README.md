# Module 8.16 of DevOps Bootcamp by [TechWorld with Nana](https://www.techworld-with-nana.com/)
Dynamically Increment Application Version in Jenkins Pipeline

## Technologies used:
- Jenkins
- Docker
- GitLab
- Git
- Java
- Maven

## Project Description
- Configure CI step: Increment patch version
- Configure CI step: Build Java application and clean old artifacts
- Configure CI step: Build image with dynamic Docker image tag
- Configure CI step: Push image to private Docker Hub repository
- Configure CI step: Commit version update of Jenkins back to Git repository
- Configure Jenkins pipeline to not trigger automatically on CI build commit to avoid commit loop

## Implementation Steps:
1. Increment version in 'pom.xml' file of [demo project](https://github.com/felix-karg/java-maven-app) via maven plugin:
   `mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit`
   1. `build-helper:parse-version`: extracts separte values for major, minor and incremental version from actual version number
   2. `-DnewVersion=`: set new version
   3. `parsedVersion`: object that contains properties for major, minor and incremental version
   4. Available values to set are `majorVersion` (actually set major version), `nextMajorVersion` (actually set major version + 1), `minorVersion`, `nextMinorVersion`, `incrementalVersion` and `nextIncrementalVersion`
   5. `versions:commit`: apply change to 'pom.xml'
2. 
