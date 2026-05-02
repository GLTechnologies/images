https://cdn.jsdelivr.net/gh/GLTechnologies/images/

# GitLab

## 生成密码

```bash
cat /etc/netplan/50-cloud-init.yaml
sudo ip route del default via 192.168.124.1 && sudo ip route add default via 192.168.124.100 dev ens18
```



```bash
sudo docker exec -it gitlab cat /etc/gitlab/initial_root_password
```

## 一、

![image-20251229180857517](F:\picture\gitlab1.png)

## 二、创建项目 & 添加 webhook

![gitlab2](F:\picture\gitlab2.png)

![gitlab3](F:\picture\gitlab3.png)

## 三、创建 Jenkinsfile

将 Dockerfile 与 Jenkinsfile 放入项目的 ci 文件夹中

Dockerfile

```tex
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY ruoyi-admin/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

Jenkinsfile

```tex
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        HARBOR_URL = "192.168.124.17"
        HARBOR_PROJECT = "images"
        IMAGE_NAME = "ruoyi-admin"

        BUILD_TAG = "build-${BUILD_NUMBER}"
        LATEST_TAG = "latest"

        FULL_IMAGE = "${HARBOR_URL}/${HARBOR_PROJECT}/${IMAGE_NAME}"
        HARBOR_CREDENTIALS = "harbor-cred"
    }

    stages {
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                docker build -f ci/backend/Dockerfile \
                  -t ${FULL_IMAGE}:${BUILD_TAG} \
                  -t ${FULL_IMAGE}:${LATEST_TAG} \
                  .
                """
            }
        }

        stage('Docker Login Harbor.') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${HARBOR_CREDENTIALS}",
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {
                    sh """
                    echo "${HARBOR_PASS}" | docker login ${HARBOR_URL} \
                      -u "${HARBOR_USER}" --password-stdin
                    """
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh """
                docker push ${FULL_IMAGE}:${BUILD_TAG}
                docker push ${FULL_IMAGE}:${LATEST_TAG}
                """
            }
        }
    }

    post {
        always {
            sh '''
            docker logout ${HARBOR_URL} || true
            docker image prune -f || true
            '''
        }
        success {
            echo "✅ 镜像推送成功：${FULL_IMAGE}:${LATEST_TAG}"
        }
        failure {
            echo "❌ 构建或推送失败"
        }
    }
}

```



# Jenkins

## 生成密码

```bash
sudo docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

## 一、安装插件

![jenkins1](F:\picture\jenkins1.png)

安装插件：Maven Integration、Stage View、Gitlab

## 二、全局工具配置

### 1、安装Maven

![jenkins2](F:\picture\jenkins2.png)

## 三、凭证管理

![jenkins3](F:\picture\jenkins3.png)

Harbor账号密码

ID填写：harbor-cred

## 四、新建流水线-后端

![jenkins4](F:\picture\jenkins4.png)

![jenkins5](F:\picture\jenkins5.png)

脚本路径：ci/Jenkinsfile-backend

## 五、新建流水线-前端