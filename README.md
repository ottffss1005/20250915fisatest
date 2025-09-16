#  JENKINS를 사용한 CI/CD

## 프로젝트 개요

Spring/Gradle 프로젝트를 Jenkins에서 빌드하고, Docker 컨테이너에서 실행

주요 목표:

* STS에서 프로젝트 생성 후 GitHub 업로드
* Jenkins에서 자동 빌드
* Docker 컨테이너에서 bind mount 또는 volume mount를 통해 jar 실행

---

## 1️⃣ 프로젝트 생성 및 GitHub 업로드

1. STS에서 `step04_gradleBuild` 프로젝트 생성

   * GET, POST 방식 처리 메소드 구현
2. GitHub에 소스코드 push
3. Jenkins에서 `step03_teamArt`라는 Item 생성 후 Pipeline 설정

---

## 2️⃣ Jenkins Pipeline 설정

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/ottffss1005/20250915fisatest.git'
                echo "**********"
                sh 'ls -al'
                echo "**********"
            }
        }
        stage('Build') {
            steps {
                script {
                    // gradlew 실행 권한 부여 후 빌드
                    sh 'chmod +x gradlew'
                    sh './gradlew clean build'
                }
            }
        }
    }
    post {
        success {
            echo "✅ 빌드 성공!"
        }
        failure {
            echo "❌ 빌드 실패! 오류 확인 필요!"
        }
    }
}
```

* Jenkins 콘솔에서 빌드 후 `build/libs/*.jar` 확인 가능

---

## 3️⃣ Docker 컨테이너에서 실행

### 1️⃣ Bind Mount 방식

```bash
docker run -it \
  --name myubuntu \
  -v /var/jenkins_home/workspace/step03_teamArt:/app \
  ubuntu:20.04 \
  bash

# 컨테이너 내부
apt update
apt install -y openjdk-11-jdk
cd /app/build/libs
java -jar step04_gradeBuild-0.0.1-SNAPSHOT.jar
```

> ⚠️ 주의

* 호스트의 `/var/jenkins_home/workspace/step03_teamArt`가 Jenkins 컨테이너 내부의 volume이 아닐 경우 jar 파일이 보이지 않을 수 있음
* 이때는 Jenkins 컨테이너에서 빌드 후 호스트로 파일 복사 필요

```bash
docker cp myjenkins:/var/jenkins_home/workspace/step03_teamArt/build/libs /home/ubuntu/step03_teamArt_build
```

---

### 2️⃣ Docker Volume 방식

1. Volume 생성

```bash
docker volume create fisavol
docker volume ls
```

2. 빈 Volume에 jar 복사

```bash
docker run --rm \
  -v fisavol:/app \
  -v /home/ubuntu/step03_teamArt_build:/src \
  ubuntu:20.04 \
  bash -c "cp /src/*.jar /app/"
```

3. 컨테이너에서 jar 실행

```bash
docker run -it -v fisavol:/app ubuntu:20.04 bash
apt update
apt install -y openjdk-17-jdk

cd /app
java -jar step04_gradeBuild-0.0.1-SNAPSHOT.jar
```

> ⚠️ 주의

* Volume은 빈 공간이므로 자동으로 호스트 파일이 들어가지 않음
* 반드시 `cp` 명령 등으로 jar 파일을 Volume에 넣어야 실행 가능

---

## 4️⃣ 트러블슈팅

| 문제                     | 원인                                                | 해결                            |
| ---------------------- | ------------------------------------------------- | ----------------------------- |
| Jenkins 빌드 후 jar가 안 보임 | 호스트 경로를 bind mount 했지만 Jenkins 컨테이너 내부 Volume이 아님 | `docker cp`로 빌드 결과를 호스트로 복사   |
| Docker Volume에 jar 없음  | Volume은 빈 공간                                      | `docker run -v cp`로 파일 복사 |
| Gradle 실행 실패           | 실행 권한 없음                                          | `chmod +x gradlew` 추가         |

### Bind Mount에서 jar 파일이 보이지 않을 때

Jenkins에서 `step03_teamArt` 프로젝트를 빌드했는데, 컨테이너에서 실행한 bind mount 경로 `/app`에 jar 파일이 보이지 않음.

```
/var/jenkins_home/workspace/step03_teamArt
├── src
├── build
│   └── libs
│       └── step04_gradeBuild-0.0.1-SNAPSHOT.jar
├── gradlew
├── build.gradle
└── ...
```

### 원인 분석

1. 호스트 `/var/jenkins_home/workspace/step03_teamArt` 폴더는 **Jenkins 컨테이너 내부의 volume이 아닌 단순한 호스트 폴더**였음.
2. `docker run -v /var/jenkins_home/workspace/step03_teamArt:/app`를 통해 bind mount를 시도했지만, **호스트 폴더가 비어있으면 컨테이너 내부 빌드 결과가 덮어씌워져 보이지 않음**.
3. Jenkins 컨테이너 내부에서 생성된 jar 파일은 **컨테이너의 볼륨 내부**에만 존재하고, 호스트 폴더에는 자동으로 복사되지 않음.

### 해결 방법

1. 컨테이너 내부에서 호스트로 빌드 결과 복사

```bash
docker cp myjenkins:/var/jenkins_home/workspace/step03_teamArt/build/libs /home/ubuntu/step03_teamArt_build
```

> <img width="998" height="125" alt="Image" src="https://github.com/user-attachments/assets/145fdfee-2bac-4463-b7df-e539503d1f60" />


> <img width="1381" height="485" alt="Image" src="https://github.com/user-attachments/assets/3140717c-4b27-492d-bddd-38c09b85135b" />

---
## 5️⃣ 경로

- Jenkins Home: /var/jenkins_home
- 프로젝트 Workspace: /var/jenkins_home/workspace/step03_teamArt
- 빌드 결과 jar: /build/libs/step04_gradeBuild-0.0.1-SNAPSHOT.jar
