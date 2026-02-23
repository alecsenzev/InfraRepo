node('Nikita_label') {
    stage('Deploy') {
        sh '''
            echo "=== НАЧАЛО ДЕПЛОЯ ==="
            
            # Параметры
            TARGET_IP="192.168.199.161"
            TARGET_USER="debian"
            SSH_KEY="/home/ubuntu/Nikita_Alecsentsev.pem"
            
            # Копируем артефакт из задачи Лабы 2
            echo "Копирование артефакта из My_project..."
            cp -r /var/lib/jenkins/jobs/My_project/lastSuccessful/archive/target/*.jar ./ || {
                echo "НЕ УДАЛОСЬ скопировать артефакт!"
                exit 1
            }
            
            # Проверяем, что файл скопировался
            ls -la *.jar || echo "JAR файл не найден!"
            
            # Копируем на целевой сервер
            echo "Копирование на сервер $TARGET_IP..."
            scp -i $SSH_KEY -o StrictHostKeyChecking=no *.jar $TARGET_USER@$TARGET_IP:/tmp/
            
            # Запускаем на целевом сервере
            echo "Запуск приложения на сервере..."
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no $TARGET_USER@$TARGET_IP "
                echo 'Подключились к серверу'
                
                # Создаем директорию
                sudo mkdir -p /opt/restoring-app
                sudo chown $TARGET_USER:$TARGET_USER /opt/restoring-app
                
                # Останавливаем старое приложение
                echo 'Останавливаем старое приложение...'
                pkill -f 'java.*jar' || echo 'Нет запущенного приложения'
                sleep 2
                
                # Устанавливаем новое
                echo 'Устанавливаем новое приложение...'
                mv /tmp/*.jar /opt/restoring-app/
                cd /opt/restoring-app
                
                # Запускаем
                echo 'Запускаем новое приложение...'
                nohup java -jar *.jar > ~/app.log 2>&1 &
                sleep 3
                
                # Проверяем
                echo 'Проверка запуска:'
                ps aux | grep java | grep -v grep || echo 'Java процесс НЕ ЗАПУЩЕН!'
                echo 'Лог приложения:'
                tail -5 ~/app.log 2>/dev/null || echo 'Лог пока пуст'
            "
            
            echo "=== ДЕПЛОЙ ЗАВЕРШЕН ==="
        '''
    }
}
