pipeline {
    agent any
    
    environment {
        APP_REPO = 'https://github.com/alecsenzev/SimpleApp.git'
        APP_BRANCH = 'main'
        STACK_NAME = 'Nikita-stack'
        HEAT_TEMPLATE = 'server.yaml'
    }
    
    stages {
        stage('Checkout InfraRepo') {
            steps {
                git branch: 'main', url: 'https://github.com/alecsenzev/InfraRepo.git'
            }
        }
        
        stage('Create Infrastructure via Heat') {
            steps {
                script {
                    sh """
                        . /home/ubuntu/openrc.sh
                        
                        # Проверяем существование стека и удаляем если есть
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
                        
                        # Если не нашли IP через regex, пробуем другой способ
                        if [ -z "\$SERVER_IP" ]; then
                            SERVER_IP=\$(openstack server show \$SERVER_ID -f shell | grep "^addresses=" | cut -d'=' -f2 | tr -d '"' | grep -oE '[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}' | head -1)
                        fi
                        
                        # Проверяем что IP получен
                        if [ -z "\$SERVER_IP" ]; then
                            echo "ОШИБКА: Не удалось получить IP адрес сервера"
                            exit 1
                        fi
                        
                        echo "SERVER_IP=\$SERVER_IP"
                        echo \$SERVER_IP > server_ip.txt
                    """
                    
                    // Читаем IP из файла и сохраняем в переменную
                    def serverIp = readFile('server_ip.txt').trim()
                    echo "✅ IP адрес сервера: ${serverIp}"
                    
                    // Сохраняем IP для следующих стадий
                    env.TARGET_SERVER_IP = serverIp
                }
            }
        }
        
        stage('Deploy Application on Target Server') {
            steps {
                script {
                    sh """
                        echo "Ожидаем доступность сервера ${env.TARGET_SERVER_IP} по SSH..."
                        
                        # Ждем пока сервер станет доступен (до 2 минут)
                        for i in {1..12}; do
                            if nc -z -w 5 ${env.TARGET_SERVER_IP} 22 2>/dev/null; then
                                echo "✅ Сервер доступен по SSH"
                                break
                            fi
                            echo "Попытка \$i из 12: сервер еще не доступен, ждем 10 сек..."
                            sleep 10
                        done
                        
                        # Клонируем репозиторий приложения и деплоим
                        ssh -o StrictHostKeyChecking=no ubuntu@${env.TARGET_SERVER_IP} '
                            # Клонируем репозиторий
                            if [ -d ~/SimpleApp ]; then
                                rm -rf ~/SimpleApp
                            fi
                            
                            git clone ${APP_REPO} ~/SimpleApp
                            cd ~/SimpleApp
                            
                            # Устанавливаем зависимости и запускаем
                            pip3 install --user -r requirements.txt
                            
                            # Останавливаем предыдущий процесс если был
                            pkill -f app.py || true
                            
                            # Запускаем приложение в screen
                            screen -dm bash -c "python3 app.py"
                            
                            echo "✅ Приложение запущено"
                        '
                    """
                }
            }
        }
        
        stage('Run Application & Health Checks') {
            steps {
                script {
                    // Ждем пока приложение запустится
                    sleep(15)
                    
                    // Проверяем здоровье приложения
                    sh """
                        echo "Проверка здоровья приложения на ${env.TARGET_SERVER_IP}:8000..."
                        
                        # Проверяем доступность приложения (до 30 сек)
                        for i in {1..6}; do
                            HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://${env.TARGET_SERVER_IP}:8000/health 2>/dev/null || echo "000")
                            
                            if [ "\$HTTP_CODE" = "200" ]; then
                                echo "✅ Приложение работает, HTTP код: \$HTTP_CODE"
                                curl -s http://${env.TARGET_SERVER_IP}:8000/health
                                echo ""
                                break
                            else
                                echo "Попытка \$i из 6: приложение еще не отвечает (HTTP \$HTTP_CODE), ждем 5 сек..."
                                sleep 5
                            fi
                        done
                        
                        # Основная проверка
                        curl -f http://${env.TARGET_SERVER_IP}:8000/health || exit 1
                    """
                    
                    echo "✅ Все проверки здоровья пройдены успешно!"
                }
            }
        }
        
        stage('Collect Artifacts') {
            steps {
                script {
                    // Получаем текущую дату в Groovy
                    def currentDate = new Date().format('yyyy-MM-dd HH:mm:ss')
                    
                    // Собираем информацию о деплое
                    sh """
                        . /home/ubuntu/openrc.sh
                        
                        # Получаем информацию о стеке
                        STACK_STATUS=\$(openstack stack show $STACK_NAME -f value -c stack_status)
                        
                        # Получаем информацию о сервере
                        SERVER_ID=\$(openstack stack resource list $STACK_NAME -f value -c physical_resource_id | head -1)
                        SERVER_STATUS=\$(openstack server show \$SERVER_ID -f value -c status)
                        
                        # Создаем файл с информацией о деплое (используем echo вместо heredoc)
                        echo "Деплой приложения на ${currentDate}" > deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "Информация о сервере:" >> deployment-info.txt
                        echo "- IP адрес: ${env.TARGET_SERVER_IP}" >> deployment-info.txt
                        echo "- ID сервера: \$SERVER_ID" >> deployment-info.txt
                        echo "- Статус сервера: \$SERVER_STATUS" >> deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "Информация о стеке Heat:" >> deployment-info.txt
                        echo "- Имя стека: $STACK_NAME" >> deployment-info.txt
                        echo "- Статус стека: \$STACK_STATUS" >> deployment-info.txt
                        echo "" >> deployment-info.txt
                        echo "Информация о приложении:" >> deployment-info.txt
                        echo "- URL приложения: http://${env.TARGET_SERVER_IP}:8000" >> deployment-info.txt
                        echo "- Health check: http://${env.TARGET_SERVER_IP}:8000/health" >> deployment-info.txt
                        
                        echo "✅ Информация сохранена в deployment-info.txt"
                        cat deployment-info.txt
                    """
                    
                    // Сохраняем артефакты
                    archiveArtifacts artifacts: 'deployment-info.txt', fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            script {
                // Очищаем порты
                sh """
                    fuser -k 8000/tcp 2>/dev/null || true
                    fuser -k 8050/tcp 2>/dev/null || true
                    fuser -k 8051/tcp 2>/dev/null || true
                """
                
                // Выводим информацию о завершении
                if (currentBuild.currentResult == 'SUCCESS') {
                    echo "✅ Сборка успешно завершена!"
                    if (env.TARGET_SERVER_IP) {
                        echo "Приложение доступно по адресу: http://${env.TARGET_SERVER_IP}:8000"
                    }
                } else {
                    echo "❌ Сборка завершилась с ошибкой: ${currentBuild.currentResult}"
                    echo "Проверьте логи выше для деталей."
                }
            }
        }
        
        failure {
            echo "❌ Деплой не удался. Проверьте логи и повторите попытку."
        }
        
        success {
            echo "🎉 Деплой успешно завершен!"
        }
    }
}
