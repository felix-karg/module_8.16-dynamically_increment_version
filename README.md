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

## Related Repository:
- [Demo project](https://github.com/felix-karg/java-maven-app) (Branch 'module_8.16-increment_version')

## Implementation Steps:
1. Increment version in 'pom.xml' file of [demo project](https://github.com/felix-karg/java-maven-app) via maven plugin:
   `mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextIncrementalVersion} versions:commit`
   1. `build-helper:parse-version`: extracts separte values for major, minor and incremental version from actual version number
   2. `-DnewVersion=`: set new version
   3. `parsedVersion`: object that contains properties for major, minor and incremental version
   4. Available values to set are `majorVersion` (actually set major version), `nextMajorVersion` (actually set major version + 1), `minorVersion`, `nextMinorVersion`, `incrementalVersion` and `nextIncrementalVersion`
   5. `versions:commit`: apply change to 'pom.xml'
   6. Thats the syntax for Windows CMD. For Unix-like shells you need to escape `$` with a preceeding `\`
2. To increment version during pipeline execution add another stage right before the stage that builds the application in Jenkinsfile like this:
   ```
   stage('increment version') {
         steps {
             script {
                 echo 'incrementing app version...'
                 sh 'mvn build-helper:parse-version versions:set \
                     -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                     versions:commit'
             }
         }
     }
   ```
3. To apply the actual version to the docker image name dynamically during pipeline execution we first need to use a variable for image name like this in the Jenkinsfile:
   ```
   sh "docker build -t <docker-hub-repo>/demo-app:$IMAGE_NAME ."
   sh "docker push  <docker-hub-repo>/demo-app:$IMAGE_NAME"
   ```
4. At the end of the script block of 'increment version' stage from step 2 above add the following lines:
   ```
   def matcher = readFile('pom.xml') =- '<version>(.*)</version>'
   def version = matcher[0][1]
   IMAGE_NAME = "$version-$BUILD_NUMBER"
   ```
   First line returns an array with all occurances of regex '<version>something_within</version>' from 'pom.xml' file
   Second line extracts first element of that array (the first occurrance of version node) which is another array, and from that the second element, which is the content of the 'version' tags
   Third line appends the Jenkins build number and saves to variable 'IMAGE_NAME'
6. In Dockerfile instead of `ENTRYPOINT` there needs to be used `CMD` to allow regular expression in command like this:
   ```
   CMD java -jar java-maven-app-*.jar
   ```
7. To avoid conflict between mulitible jar files for packaging there need to be executed `mvn clean package` in the pipeline script
