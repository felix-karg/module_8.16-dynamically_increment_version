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
   sh "docker build -t <docker-hub-repo>/demo-app:${IMAGE_NAME} ."
   sh "docker push  <docker-hub-repo>/demo-app:${IMAGE_NAME}"
   ```
4. At the end of the script block of 'increment version' stage from step 2 above add the following lines:
   ```
   def matcher = readFile('pom.xml') =~ '<version>(.*)</version>'
   def version = matcher[0][1]
   IMAGE_NAME = "$version-$BUILD_NUMBER"
   ```
   - First line returns an array with all occurances of regex '<version>something_within</version>' from 'pom.xml' file
   - Second line extracts first element of that array (the first occurrance of version node) which is another array, and from that the second element, which is the content of the 'version' tags
   - Third line appends the Jenkins build number and saves to variable 'IMAGE_NAME'
6. In Dockerfile instead of `ENTRYPOINT` there needs to be used `CMD` to allow regular expression in command like this:
   ```
   CMD java -jar java-maven-app-*.jar
   ```
7. To avoid conflict between mulitible jar files for packaging there need to be executed `mvn clean package` in the pipeline script
8. Execute pipeline and verify that image is built an pushed to Docker Hub

The problem with the current setup is that it changes the version only locally during pipeline run and never pushes the change back to the remote repository. So at every new pipeline run
the increment of the version start again from the original version (in our case 1.0.0-SNAPSHOT) and not from the version of the previous pipeline run. So wee need further adjustments:

9. Add a new stage at the end of the Jeninfile like this:
   ```
   stage("commit version update") {
         steps {
             script {
                 withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                     sh 'git config --global user.email "jenkins@example.com"'
                     sh 'git config --global user.name "jenkins"'

                     sh 'git status'
                     sh 'git branch'
                     sh 'git config --list'

                     sh "git remote set-url origin https://x-access-token:${GITHUB_TOKEN}@github.com/felix-karg/java-maven-app.git"
                     sh 'git add .'
                     sh 'git commit -m "ci: version bump"'
                     sh 'git push origin HEAD:module_8.16-increment_version'
                 }
             }
         }
     }
   ```
   - There must be credentials 'github-token' of type 'Secret Text' set in Jenkins, that contian Personal Access Token for GitHub
   - First two git commands add required settings
   - Second three commands print logging information
   - Last four commands set remote repository, then add, commit and push to this repository
10. Run pipeline to see if the stage works well and check git repository to verify the commit done by Jenkins run

When we have configured a hook to trigger pipeline automatically at every push to the repository, we now have created an infinitive loop of pushes and pipeline runs:
Every push triggers the pipeline, which does another push for increased verion, which triggers the pipeline again. To prevent this we can do the following:

11. Install Jenkins plugin 'Ignore Committer Strategy'
12. After installation a new configuration 'Build strategies' is available for multibranch pipeline
13. Click 'Add' -> 'Ignore Committer Strategy'
14. Add emain address of Jenkins user from git config set in step 9 to be ignored by trigger
15. Check the box 'Allow builds when a changeset contains non-ignored author(s)' and save
