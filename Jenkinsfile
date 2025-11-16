// 定义全局变量，以便在 post 阶段也能访问
def REGISTRY = "192.168.10.42:8443"
def PROJECT = "jenkins"
def APP_NAME = "hello-k8s-app"
def GIT_CREDENTIALS_ID = "git-credentials"
def HARBOR_CRED_ID = "harbor-credentials"
def KUBE_CONFIG_ID = "kubeconfig-credentials"

pipeline {
    agent any  // 可改为 kubernetes agent，直接在 K8s 集群中运行构建任务

    environment {
        // 在这里定义凭证 ID 或其他不需要在 post 阶段访问的变量
        // 注意：BUILD_NUMBER 是 Jenkins 内置变量，无需定义
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "📥 拉取代码从 Git 仓库"
                git branch: 'main', url: 'https://github.com/cui1519271525/hello-k8s-app.git', credentialsId: GIT_CREDENTIALS_ID
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    echo "🔨 构建并推送 Docker 镜像到 Harbor: ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}"
                    // 验证 Docker 服务可用性
                    sh "docker --version"
                    
                    withCredentials([usernamePassword(
                        credentialsId: HARBOR_CRED_ID,
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )]) {
                        sh """
                        # 登录 Harbor（忽略自签名证书警告）
                        docker login -u ${HARBOR_USER} -p ${HARBOR_PASS} ${REGISTRY} --insecure-registry
                        
                        # 构建镜像（标签含构建号，便于版本追溯）
                        docker build -t ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER} ./hello-k8s-app
                        
                        # 推送镜像到 Harbor
                        docker push ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}
                        
                        # 标签为 latest（便于快速引用最新版本）
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
                        credentialsId: KUBE_CONFIG_ID,
                        variable: 'KUBE_CONFIG_FILE'
                    )]) {
                        sh """
                        # 配置 Kubeconfig
                        mkdir -p ~/.kube
                        cp ${KUBE_CONFIG_FILE} ~/.kube/config
                        export KUBECONFIG=~/.kube/config
                        
                        # 验证 K8s 集群连接
                        kubectl cluster-info
                        
                        # 更新 Deployment 中的镜像版本（替换为当前构建号）
                        sed -i "s|image: ${REGISTRY}/${PROJECT}/${APP_NAME}:.*|image: ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER}|" ./hello-k8s-app/k8s/deployment.yaml
                        
                        # 应用 K8s 配置（部署 Deployment 和 Service）
                        kubectl apply -f ./hello-k8s-app/k8s/deployment.yaml
                        kubectl apply -f ./hello-k8s-app/k8s/service.yaml
                        
                        # 可选：应用 Ingress 配置（需集群已部署 Ingress Controller）
                        if [ -f ./hello-k8s-app/k8s/ingress.yaml ]; then
                            kubectl apply -f ./hello-k8s-app/k8s/ingress.yaml
                        fi
                        
                        # 等待 Pod 就绪（最多等待 3 分钟）
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
                        credentialsId: KUBE_CONFIG_ID,
                        variable: 'KUBE_CONFIG_FILE'
                    )]) {
                        sh """
                        export KUBECONFIG=${KUBE_CONFIG_FILE}
                        
                        # 获取 Service 的 NodePort 和节点 IP
                        NODE_PORT=\$(kubectl get svc ${APP_NAME}-service -o jsonpath='{.spec.ports[0].nodePort}')
                        NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
                        APP_URL="http://\${NODE_IP}:\${NODE_PORT}"
                        
                        echo "应用访问地址：\${APP_URL}"
                        echo "开始测试访问（重试 10 次，每次间隔 5 秒）"
                        
                        # 测试应用可用性（含健康检查接口）
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
                credentialsId: KUBE_CONFIG_ID,
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
            // 清理本地镜像，避免占用磁盘空间
            // 使用全局变量 REGISTRY, PROJECT, APP_NAME
            sh "docker rmi ${REGISTRY}/${PROJECT}/${APP_NAME}:BUILD-${BUILD_NUMBER} || true"
            sh "docker rmi ${REGISTRY}/${PROJECT}/${APP_NAME}:latest || true"
            // 清理 Kubeconfig（如果在构建节点上创建了的话）
            sh "rm -rf ~/.kube/config || true"
        }
    }
}
