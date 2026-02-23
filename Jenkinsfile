pipeline {
    agent any
    
    environment {
        // --- Настройки для доступа к артефакту из Л2 ---
        // Имя job'ы из Л2, которая собирает приложение
        BUILD_JOB_NAME = 'Nikita-Build-App-L2' 
        // Номер сборки, из которой брать артефакт. 
        // 'lastSuccessfulBuild' - последняя успешная, 'lastCompleted' - последняя завершенная.
        // Можно также параметризовать.
        BUILD_NUMBER = 'lastSuccessfulBuild' 
        
        // --- Настройки инфраструктуры (из Л3) ---
        STACK_NAME = 'Nikita-stack-Deploy'
        HEAT_TEMPLATE = 'server.yaml' // Убедитесь, что этот файл есть в вашем репозитории InfraRepo
        APP_REPO = 'https://github.com/alecsenzev/InfraRepo.git' // Репозиторий с шаблонами
        
        // --- Настройки приложения ---
        // Тип артефакта и порт, на котором оно будет работать
        ARTIFACT_PATTERN = '**/target/*.jar' // Пример для Java (Maven). Измените под ваш проект
        APP_PORT = '8080' 
    }
    
    stages {
        stage('Checkout InfraRepo') {
            steps {
                // Забираем наш репозиторий с шаблонами Heat
                git branch: 'main', url: "${APP_REPO}"
            }
        }
        
        stage('Get Artifact from Build Job (L2)') {
            steps {
                script {
                    // Копируем артефакт из задачи Л2 в текущую рабочую директорию
                    // Это ключевой шаг!
                    copyArtifacts(
                        projectName: "${BUILD_JOB_NAME}",
                        selector: specific("${BUILD_NUMBER}"),
                        // Куда положить артефакт внутри workspace текущей задачи
                        target: 'downloaded-artifact/' 
                    )
                    
                    echo "✅ Артефакт из задачи ${BUILD_JOB_NAME} успешно скопирован."
                    
                    // Найдем имя скопированного файла, чтобы использовать позже
                    sh '''
                        cd downloaded-artifact/
                        ARTIFACT_FILE=$(ls *.jar | head -1) 
                        if [ -z "$ARTIFACT_FILE" ]; then
                            echo "ОШИБКА: Не найден JAR файл в downloaded-artifact/"
                            exit 1
                        fi
                        echo "ARTIFACT_FILE_NAME=$ARTIFACT_FILE" > ../artifact-name.txt
                    '''
                    env.ARTIFACT_FILE_NAME = readFile('artifact-name.txt').trim().replace('ARTIFACT_FILE_NAME=', '')
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
                        
                        # Копируем артефакт на сервер с помощью scp
                        # Используем пользователя ubuntu, если образ другой - измените
                        scp -o StrictHostKeyChecking=no -r downloaded-artifact/* ubuntu@${env.TARGET_SERVER_IP}:~/
                        
                        # Устанавливаем Java и другие зависимости на целевом сервере
                        # (Если ваш образ уже имеет все нужное, этот шаг можно пропустить)
                        ssh -o StrictHostKeyChecking=no ubuntu@${env.TARGET_SERVER_IP} '
                            # Установка Java, если её нет
                            if ! command -v java &> /dev/null; then
                                sudo apt update
                                sudo apt install -y openjdk-17-jre-headless
                            fi
                            
                            # Здесь можно добавить установку PostgreSQL, если нужно
                            # sudo apt install -y postgresql
                            
                            echo "✅ Сервер подготовлен"
                        '
                    """
                }
            }
        }
        
        stage('Run Application') {
            steps {
                script {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${env.TARGET_SERVER_IP} '
                            # Останавливаем предыдущий процесс, если был
                            pkill -f ${ARTIFACT_FILE_NAME} || true
                            
                            # Запускаем JAR. Флаг -Dserver.port=... нужен, если приложение использует Spring Boot
                            # Уберите его, если не нужен
                            nohup java -Dserver.port=${APP_PORT} -jar ~/${ARTIFACT_FILE_NAME} > app.log 2>&1 &
                            
                            echo "✅ Приложение запущено"
                        '
                    """
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    sleep(20) // Даем время приложению стартануть
                    
                    sh """
                        echo "Проверка здоровья приложения на ${env.TARGET_SERVER_IP}:${APP_PORT}..."
                        
                        HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://${env.TARGET_SERVER_IP}:${APP_PORT}/health 2>/dev/null || echo "000")
                        
                        if [ "\$HTTP_CODE" = "200" ] || [ "\$HTTP_CODE" = "404" ]; then
                            echo "✅ Приложение отвечает, HTTP код: \$HTTP_CODE"
                        else
                            echo "❌ Приложение не отвечает корректно, HTTP код: \$HTTP_CODE"
                            exit 1
                        fi
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
                        
                        echo "Деплой приложения \$(date)" > deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "Информация о сборке:" >> deployment-info.txt
                        echo "- Артефакт: ${env.ARTIFACT_FILE_NAME}" >> deployment-info.txt
                        echo "- Взят из job: ${BUILD_JOB_NAME} [${BUILD_NUMBER}]" >> deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "Информация о сервере:" >> deployment-info.txt
                        echo "- IP адрес: ${env.TARGET_SERVER_IP}" >> deployment-info.txt
                        echo "- Стек Heat: $STACK_NAME" >> deployment-info.txt
                        echo "- Статус стека: \$STACK_STATUS" >> deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "URL приложения: http://${env.TARGET_SERVER_IP}:${APP_PORT}"
                    """
                    
                    archiveArtifacts artifacts: 'deployment-info.txt'
                }
            }
        }
    }
    
    post {
        success {
            echo "🎉 Деплой артефакта из Л2 на инфраструктуру из Л3 успешно завершен!"
            echo "Приложение доступно по адресу: http://${env.TARGET_SERVER_IP}:${APP_PORT}"
        }
        failure {
            echo "❌ Деплой не удался."
        }
    }
}
