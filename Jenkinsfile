// 定义全局变量，以便在 post 阶段也能访问
def REGISTRY = "192.168.10.42:8443"
def PROJECT = "jenkins"
def APP_NAME = "hello-k8s-app"

pipeline {
    agent any

    // 在 environment 块中定义凭证 ID 或其他环境变量
    environment {
        GIT_CREDENTIALS_ID = "git-credentials"
        HARBOR_CREDENTIALS_ID = "harbor-credentials"
        KUBE_CONFIG_CREDENTIALS_ID = "kubeconfig-credentials"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📥 拉取代码从 Git 仓库"
                // 使用 environment 块中定义的凭证 ID
                git branch: 'main', url: 'https://github.com/cui1519271525/hello-k8s-app.git', credentialsId: "${GIT_CREDENTIALS_ID}"
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "🔨 构建并推送 Docker 镜像到 Harbor: ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}"
                    
                    // 使用 withCredentials 绑定凭证，并引用 environment 块中定义的 ID
                    withCredentials([usernamePassword(
                        credentialsId: "${HARBOR_CREDENTIALS_ID}",
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                        # 登录 Harbor（忽略自签名证书警告）
                        docker login -u ${HARBOR_USER} -p ${HARBOR_PASS} ${REGISTRY} --insecure-registry
                        
                        # 构建镜像
                        docker build -t ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER} .
                        
                        # 推送镜像
                        docker push ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}
                        
                        # 标签为 latest
                        docker tag ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER} ${REGISTRY}/${PROJECT}/${APP_NAME}:latest
                        docker push ${REGISTRY}/${PROJECT}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "🚀 部署应用到 Kubernetes 集群"
                    withCredentials([file(
                        credentialsId: "${KUBE_CONFIG_CREDENTIALS_ID}",
                        variable: 'KUBE_CONFIG_FILE'
                    )]) {
                        sh """
                        # 配置 Kubeconfig
                        mkdir -p ~/.kube
                        cp ${KUBE_CONFIG_FILE} ~/.kube/config
                        export KUBECONFIG=~/.kube/config
                        
                        # 验证 K8s 集群连接
                        kubectl cluster-info
                        
                        # 更新 Deployment 镜像
                        sed -i "s|image: ${REGISTRY}/${PROJECT}/${APP_NAME}:.*|image: ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}|" ./k8s/deployment.yaml
                        
                        # 应用 K8s 配置
                        kubectl apply -f ./k8s/deployment.yaml
                        kubectl apply -f ./k8s/service.yaml
                        
                        # 等待 Pod 就绪
                        kubectl wait --for=condition=ready pod -l app=${APP_NAME} --timeout=3m
                        """
                    }
                }
            }
        }

        stage('Test Deployment') {
            steps {
                script {
                    echo "✅ 验证应用部署结果"
                    withCredentials([file(
                        credentialsId: "${KUBE_CONFIG_CREDENTIALS_ID}",
                        variable: 'KUBE_CONFIG_FILE'
                    )]) {
                        sh """
                        export KUBECONFIG=${KUBE_CONFIG_FILE}
                        
                        # 获取 Service 地址
                        NODE_PORT=\$(kubectl get svc ${APP_NAME}-service -o jsonpath='{.spec.ports[0].nodePort}')
                        NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
                        APP_URL="http://\${NODE_IP}:\${NODE_PORT}"
                        
                        echo "应用访问地址：\${APP_URL}"
                        
                        # 测试访问
                        curl --retry 10 --retry-delay 5 --retry-connrefused \${APP_URL}
                        curl --retry 10 --retry-delay 5 --retry-connrefused \${APP_URL}/health
                        
                        echo "🎉 应用测试通过！"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "🎊 流水线执行成功！"
            withCredentials([file(
                credentialsId: "${KUBE_CONFIG_CREDENTIALS_ID}",
                variable: 'KUBE_CONFIG_FILE'
            )]) {
                sh """
                export KUBECONFIG=${KUBE_CONFIG_FILE}
                NODE_PORT=\$(kubectl get svc ${APP_NAME}-service -o jsonpath='{.spec.ports[0].nodePort}')
                NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
                echo "最终访问地址：http://\${NODE_IP}:\${NODE_PORT}"
                """
            }
        }
        failure {
            echo "❌ 流水线执行失败，请查看日志排查问题。"
        }
        always {
            echo "🧹 清理构建环境"
            // 清理本地镜像，这里使用了全局变量
            sh "docker rmi ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER} || true"
            sh "docker rmi ${REGISTRY}/${PROJECT}/${APP_NAME}:latest || true"
            // 清理 Kubeconfig
            sh "rm -rf ~/.kube/config || true"
        }
    }
}
