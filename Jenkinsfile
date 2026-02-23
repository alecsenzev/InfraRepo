node('Nikita_label') {
    stage('Get Artifact') {
        step([$class: 'CopyArtifact', 
              projectName: 'My_project', 
              filter: 'target/*.jar', 
              target: ''])
    }
    
    stage('Deploy') {
        sh '''
            # Переменные
            TARGET_IP="192.168.199.161"
            TARGET_USER="debian"
            SSH_KEY="/home/ubuntu/Nikita_Alecsentsev.pem"
            
            # Находим JAR файл
            JAR_FILE=$(ls target/*.jar | head -1)
            echo "Найден артефакт: $JAR_FILE"
            
            # Копируем на целевой сервер
            echo "Копирование на сервер $TARGET_IP..."
            scp -i $SSH_KEY -o StrictHostKeyChecking=no $JAR_FILE $TARGET_USER@$TARGET_IP:/tmp/
            
            # Запускаем на целевом сервере
            echo "Запуск приложения..."
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no $TARGET_USER@$TARGET_IP "
                sudo mkdir -p /opt/restoring-app
                sudo chown $TARGET_USER:$TARGET_USER /opt/restoring-app
                
                pkill -f 'java.*jar' || true
                sleep 2
                
                mv /tmp/*.jar /opt/restoring-app/
                cd /opt/restoring-app
                
                nohup java -jar *.jar > ~/app.log 2>&1 &
                sleep 3
                
                echo 'Приложение запущено с PID:'
                pgrep -f 'java.*jar'
            "
        '''
    }
    
    stage('Verify') {
        sh '''
            ssh -i /home/ubuntu/Nikita_Alecsentsev.pem -o StrictHostKeyChecking=no debian@192.168.199.161 "tail -5 ~/app.log || echo 'Лог пока пуст'"
        '''
    }
}
