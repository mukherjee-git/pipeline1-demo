# Java Maven CI/CD Demo (Jenkins → Sonar → Docker → Nexus/JFrog → Argo CD)

This repo is a realistic sample Java/Spring Boot service you can wire into Jenkins.

## Pipeline stages (Jenkinsfile)
- Maven build
- Unit tests (Surefire)
- Functional tests (Failsafe, *IT.java)
- Sonar scan + Quality Gate
- Docker build
- Performance test (k6 script against a running container)
- Push image to Nexus + JFrog
- Update GitOps repo overlays (nexus + jfrog)
- Argo CD sync (two Applications)

## Run locally
```bash
mvn clean test
mvn -Pfunctional-tests verify
mvn spring-boot:run
curl http://localhost:8080/api/hello
```
