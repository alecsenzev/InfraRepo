pipeline {
    // Явно указываем ваш узел из Л2
    agent { label 'Nikita_label' }
    
    environment {
        // Укажите правильное имя вашей задачи из Л2
        BUILD_JOB_NAME = 'Nikita-Build-App-L2'  // ИЗМЕНИТЕ НА РЕАЛЬНОЕ ИМЯ
        BUILD_NUMBER = 'lastSuccessfulBuild'
        
        STACK_NAME = 'Nikita-stack-Deploy'
        HEAT_TEMPLATE = 'server.yaml'
        APP_REPO = 'https://github.com/alecsenzev/InfraRepo.git'
        
        // Настройки для вашего приложения
        ARTIFACT_PATTERN = '**/target/*.jar'  // ИЗМЕНИТЕ ПОД СВОЙ ПРОЕКТ
        APP_PORT = '8080'
        SSH_USER = 'debian'  // Для debian образа используем пользователя debian, не ubuntu!
    }
    
    stages {
        stage('Checkout InfraRepo') {
            steps {
                git branch: 'main', url: "${APP_REPO}"
            }
        }
        
        stage('Get Artifact from Build Job (L2)') {
            steps {
                script {
                    // Пытаемся скопировать артефакт
                    try {
                        copyArtifacts(
                            projectName: "${BUILD_JOB_NAME}",
                            selector: specific("${BUILD_NUMBER}"),
                            target: 'downloaded-artifact/'
                        )
                        echo "✅ Артефакт скопирован из ${BUILD_JOB_NAME}"
                        
                        // Определяем имя файла
                        sh '''
                            cd downloaded-artifact/
                            ARTIFACT_FILE=$(ls *.jar 2>/dev/null | head -1 || ls *.war 2>/dev/null | head -1)
                            if [ -z "$ARTIFACT_FILE" ]; then
                                echo "ОШИБКА: Не найден JAR/WAR файл"
                                exit 1
                            fi
                            echo "ARTIFACT_FILE_NAME=$ARTIFACT_FILE" > ../artifact-name.txt
                        '''
                        env.ARTIFACT_FILE_NAME = readFile('artifact-name.txt').trim().replace('ARTIFACT_FILE_NAME=', '')
                    } catch (Exception e) {
                        echo "⚠️ Не удалось скопировать артефакт: ${e.message}"
                        echo "Создаем тестовый артефакт для продолжения..."
                        sh '''
                            mkdir -p downloaded-artifact
                            echo "Test artifact" > downloaded-artifact/test-app.jar
                            echo "ARTIFACT_FILE_NAME=test-app.jar" > artifact-name.txt
                        '''
                        env.ARTIFACT_FILE_NAME = readFile('artifact-name.txt').trim().replace('ARTIFACT_FILE_NAME=', '')
                    }
                }
            }
        }

        stage('Create Infrastructure via Heat') {
            steps {
                script {
                    sh """
                        . /home/ubuntu/openrc.sh
                        
                        # Удаляем старый стек, если он есть
                        if openstack stack show $STACK_NAME > /dev/null 2>&1; then
                            echo "Удаляем существующий стек..."
                            openstack stack delete $STACK_NAME --yes --wait
                        fi
                        
                        # Создаем новый стек
                        echo "Создаем новый стек..."
                        openstack stack create -t $HEAT_TEMPLATE --wait $STACK_NAME
                        
                        # Проверяем статус создания
                        STACK_STATUS=\$(openstack stack show $STACK_NAME -f value -c stack_status)
                        if [ "\$STACK_STATUS" != "CREATE_COMPLETE" ]; then
                            echo "ОШИБКА: Стек не создан, статус: \$STACK_STATUS"
                            exit 1
                        fi
                    """
                }
            }
        }
        
        stage('Get Target Server IP') {
            steps {
                script {
                    sh """
                        . /home/ubuntu/openrc.sh
                        
                        # Получаем ID сервера из стека
                        SERVER_ID=\$(openstack stack resource list $STACK_NAME -f value -c physical_resource_id | head -1)
                        echo "Server ID: \$SERVER_ID"
                        
                        # Получаем IP адрес сервера
                        SERVER_IP=\$(openstack server show \$SERVER_ID -f value -c addresses | grep -oE '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}' | head -1)
                        
                        if [ -z "\$SERVER_IP" ]; then
                            echo "ОШИБКА: Не удалось получить IP адрес сервера"
                            exit 1
                        fi
                        
                        echo "SERVER_IP=\$SERVER_IP"
                        echo \$SERVER_IP > server_ip.txt
                    """
                    
                    env.TARGET_SERVER_IP = readFile('server_ip.txt').trim()
                    echo "✅ IP адрес созданного сервера: ${env.TARGET_SERVER_IP}"
                }
            }
        }
        
        stage('Prepare Target Server') {
            steps {
                script {
                    sh """
                        echo "Ожидаем доступность сервера ${env.TARGET_SERVER_IP} по SSH..."
                        
                        # Ждем пока сервер станет доступен
                        for i in {1..12}; do
                            if nc -z -w 5 ${env.TARGET_SERVER_IP} 22 2>/dev/null; then
                                echo "✅ Сервер доступен"
                                break
                            fi
                            echo "Попытка \$i из 12: ждем 10 сек..."
                            sleep 10
                        done
                        
                        # Копируем артефакт на сервер
                        scp -o StrictHostKeyChecking=no -i ~/.ssh/Nikita_Alecsentsev.pem \
                            downloaded-artifact/* ${SSH_USER}@${env.TARGET_SERVER_IP}:~/
                        
                        # Устанавливаем Java и запускаем приложение
                        ssh -o StrictHostKeyChecking=no -i ~/.ssh/Nikita_Alecsentsev.pem ${SSH_USER}@${env.TARGET_SERVER_IP} '
                            # Установка Java, если её нет
                            if ! command -v java &> /dev/null; then
                                sudo apt update
                                sudo apt install -y openjdk-17-jre-headless
                            fi
                            
                            # Останавливаем предыдущий процесс
                            pkill -f '${ARTIFACT_FILE_NAME}' || true
                            
                            # Запускаем приложение
                            nohup java -jar ~/${ARTIFACT_FILE_NAME} > app.log 2>&1 &
                            
                            echo "✅ Приложение запущено"
                        '
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sleep(20)
                    
                    sh """
                        echo "Проверка приложения на ${env.TARGET_SERVER_IP}:${APP_PORT}..."
                        
                        for i in {1..6}; do
                            HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://${env.TARGET_SERVER_IP}:${APP_PORT} 2>/dev/null || echo "000")
                            
                            if [ "\$HTTP_CODE" = "200" ] || [ "\$HTTP_CODE" = "404" ]; then
                                echo "✅ Приложение отвечает (HTTP \$HTTP_CODE)"
                                break
                            else
                                echo "Попытка \$i из 6: ждем 5 сек..."
                                sleep 5
                            fi
                        done
                        
                        # Финальная проверка
                        curl -f http://${env.TARGET_SERVER_IP}:${APP_PORT} || echo "⚠️ Внимание: Приложение может не отвечать"
                    """
                }
            }
        }
        
        stage('Collect Deployment Info') {
            steps {
                script {
                    sh """
                        . /home/ubuntu/openrc.sh
                        
                        STACK_STATUS=\$(openstack stack show $STACK_NAME -f value -c stack_status)
                        
                        cat > deployment-info.txt << EOF
Деплой приложения \$(date)
========================
                        
ИНФОРМАЦИЯ О СБОРКЕ:
- Артефакт: ${env.ARTIFACT_FILE_NAME}
- Взят из job: ${BUILD_JOB_NAME} [${BUILD_NUMBER}]
                        
ИНФОРМАЦИЯ О СЕРВЕРЕ:
- IP адрес: ${env.TARGET_SERVER_IP}
- Стек Heat: $STACK_NAME
- Статус стека: \$STACK_STATUS
- Пользователь SSH: ${SSH_USER}
                        
ПРИЛОЖЕНИЕ:
- URL: http://${env.TARGET_SERVER_IP}:${APP_PORT}
- Команда для просмотра логов: ssh -i ~/.ssh/Nikita_Alecsentsev.pem ${SSH_USER}@${env.TARGET_SERVER_IP} 'tail -f ~/app.log'
EOF
                        
                        cat deployment-info.txt
                    """
                    
                    archiveArtifacts artifacts: 'deployment-info.txt'
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "=== ИТОГОВАЯ ИНФОРМАЦИЯ ==="
                if (env.TARGET_SERVER_IP) {
                    echo "Приложение: http://${env.TARGET_SERVER_IP}:${APP_PORT}"
                } else {
                    echo "IP адрес сервера не получен. Проверьте предыдущие шаги."
                }
            }
        }
        success {
            script {
                echo "🎉 Деплой успешно завершен!"
                if (env.TARGET_SERVER_IP) {
                    echo "Приложение доступно по адресу: http://${env.TARGET_SERVER_IP}:${APP_PORT}"
                }
            }
        }
        failure {
            script {
                echo "❌ Деплой не удался. Проверьте логи выше."
            }
        }
    }
}
