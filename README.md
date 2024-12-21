# 특이사항
AWS EC2의 관련 학습이 선행되어야 합니다 하기 글들을 확인해 주세요. <br>
[[aws] ec2를 통해 백엔드 api 서버 배포하기 (1)](https://sbi1024.github.io/devops/aws/1) <br>
[[aws] ec2를 통해 백엔드 api 서버 배포하기 (2)](https://sbi1024.github.io/devops/aws/2) <br>

# 개요
[[pipeline] 백엔드(spring boot) 프로젝트에 다양한 CI/CD 방식 적용하기
](https://sbi1024.github.io/devops/pipeline/2) 블로그의 게시글에서 sample로 제공되는 프로젝트 입니다.

# 사용법
위 블로그 글을 읽의면서 `deploy.yml` 파일의 코드를 하기 코드를 복사/붙여넣기 하며 단계적으로 따라가보세요.

### GitHub Actions에서 EC2에 직접 접근 후 빌드 및 실행
``` yml
name: Deploy To EC2
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: SSH로 EC2에 접속하기
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }} # EC2의 주소
          username: ${{ secrets.EC2_USERNAME }} # EC2 접속 username
          key: ${{ secrets.EC2_PRIVATE_KEY }} # EC2의 Key 파일의 내부 텍스트
          script_stop: true 
          script: |
            cd /home/ubuntu/api-server-pipeline # 여기 경로는 자신의 EC2에 맞는 경로로 재작성하기
            git pull origin main
            ./gradlew clean build
            sudo fuser -k -n tcp 8080 || true 
            nohup java -jar build/libs/*SNAPSHOT.jar > ./output.log 2>&1 &
```

### GitHub Actions에서 빌드 후 EC2에 빌드 파일 실행
``` yml
name: Deploy To EC2
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Github Repository 파일 불러오기
        uses: actions/checkout@v4
      - name: JDK 17버전 설치
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - name: 테스트 및 빌드하기
        run: ./gradlew clean build
      - name: 빌드된 파일 이름 변경하기
        run: mv ./build/libs/*SNAPSHOT.jar ./project.jar
      - name: SCP로 EC2에 빌드된 파일 전송하기
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          source: project.jar
          target: /home/ubuntu/api-server-pipeline/tobe
      - name: SSH로 EC2에 접속하기
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          script_stop: true
          script: |
            rm -rf /home/ubuntu/api-server-pipeline/current
            mkdir /home/ubuntu/api-server-pipeline/current
            mv /home/ubuntu/api-server-pipeline/tobe/project.jar /home/ubuntu/api-server-pipeline/current/project.jar
            cd /home/ubuntu/api-server-pipeline/current
            sudo fuser -k -n tcp 8080 || true
            nohup java -jar project.jar > ./output.log 2>&1 & 
            rm -rf /home/ubuntu/api-server-pipeline/tobe
```
