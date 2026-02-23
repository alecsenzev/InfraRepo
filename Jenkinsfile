pipeline {
    agent {
        label 'Nikita_label'  // Используем тот же агент, что и в Л2
    }
    
    parameters {
        string(name: 'TARGET_SERVER_IP', defaultValue: '192.168.199.161', description: 'IP целевого сервера из Л3')
        string(name: 'TARGET_USER', defaultValue: 'debian', description: 'Пользователь на целевом сервере')
        string(name: 'SSH_KEY_PATH', defaultValue: '/home/ubuntu/Nikita_Alecsentsev.pem', description: 'Путь к ключу на агенте')
    }
    
    environment {
        APP_DIR = '/opt/restoring-app'
        LOG_DIR = '/home/debian/app-logs'
        JAR_PATTERN = 'target/*.jar'
    }
    
    stages {
        stage('Подготовка') {
            steps {
                echo '=== Проверка доступности целевого сервера ==='
                sh """
                    ssh -i ${params.SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                        ${params.TARGET_USER}@${params.TARGET_SERVER_IP} "echo 'Сервер доступен'"
                """
            }
        }
        
        stage('Получение артефакта') {
            steps {
                echo '=== Копирование артефакта из задачи Л2 ==='
                step([
                    $class: 'CopyArtifact',
                    projectName: 'My_project',  // Имя задачи из Л2
                    filter: 'target/*.jar',
                    target: '',  // в текущую директорию
                    fingerprintArtifacts: true
                ])
                
                script {
                    // Находим имя JAR файла
                    def jarFile = findFiles(glob: 'target/*.jar')[0]
                    if (jarFile) {
                        echo "Найден артефакт: ${jarFile.name}"
                        env.JAR_NAME = jarFile.name
                        env.JAR_PATH = jarFile.path
                    } else {
                        error "JAR файл не найден! Убедитесь, что задача My_project собрана успешно."
                    }
                }
            }
        }
        
        stage('Деплой на сервер из Л3') {
            steps {
                echo '=== Копирование JAR на целевой сервер ==='
                sh """
                    scp -i ${params.SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                        ${env.JAR_PATH} \
                        ${params.TARGET_USER}@${params.TARGET_SERVER_IP}:/tmp/
                """
                
                echo '=== Установка и запуск приложения ==='
                sh """
                    ssh -i ${params.SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                        ${params.TARGET_USER}@${params.TARGET_SERVER_IP} << 'ENDSSH'
                        # Создаем директории
                        sudo mkdir -p ${APP_DIR}
                        sudo mkdir -p ${LOG_DIR}
                        sudo chown ${params.TARGET_USER}:${params.TARGET_USER} ${APP_DIR}
                        sudo chown ${params.TARGET_USER}:${params.TARGET_USER} ${LOG_DIR}
                        
                        # Бэкап старой версии
                        TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
                        if [ -f ${APP_DIR}/*.jar ]; then
                            echo "Бэкап предыдущей версии"
                            cp ${APP_DIR}/*.jar ${APP_DIR}/backup_\${TIMESTAMP}.jar 2>/dev/null || true
                        fi
                        
                        # Устанавливаем новую версию
                        echo "Установка новой версии"
                        mv /tmp/*.jar ${APP_DIR}/
                        chmod 644 ${APP_DIR}/*.jar
                        
                        # Останавливаем старое приложение
                        echo "Остановка старого приложения"
                        pkill -f "java.*jar" || true
                        sleep 3
                        
                        # Запускаем новое
                        echo "Запуск нового приложения"
                        cd ${APP_DIR}
                        nohup java -jar *.jar > ${LOG_DIR}/app_\${TIMESTAMP}.log 2>&1 &
                        sleep 5
                        
                        # Проверка запуска
                        if pgrep -f "java.*jar" > /dev/null; then
                            echo "✅ Приложение запущено успешно"
                            echo "PID: \$(pgrep -f "java.*jar")"
                            echo "Логи: ${LOG_DIR}/app_\${TIMESTAMP}.log"
                        else
                            echo "❌ Ошибка запуска приложения"
                            exit 1
                        fi
ENDSSH
                """
            }
        }
        
        stage('Проверка') {
            steps {
                echo '=== Проверка работоспособности ==='
                sh """
                    ssh -i ${params.SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                        ${params.TARGET_USER}@${params.TARGET_SERVER_IP} \
                        "ps aux | grep java | grep -v grep || echo 'Java процесс не найден'"
                """
                
                script {
                    // Пробуем получить HTTP ответ, если приложение веб
                    try {
                        sh """
                            ssh -i ${params.SSH_KEY_PATH} -o StrictHostKeyChecking=no \
                                ${params.TARGET_USER}@${params.TARGET_SERVER_IP} \
                                "curl -s -o /dev/null -w '%{http_code}' http://localhost:8051 || echo 'Сервис не отвечает'"
                        """
                    } catch (Exception e) {
                        echo "Приложение не отвечает по HTTP, но возможно это консольное приложение"
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Деплой успешно завершен!'
            echo "Приложение запущено на: http://${params.TARGET_SERVER_IP}:8051/"
        }
        failure {
            echo '❌ Деплой завершился с ошибкой'
            // Можно добавить отправку уведомлений
        }
        always {
            // Архивируем логи деплоя
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
    }
}
