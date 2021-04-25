### jenkins



[Java SE Development Kit 16 - Downloads (oracle.com)](https://www.oracle.com/java/technologies/javase-jdk16-downloads.html)

[安装Jenkins](https://www.jenkins.io/zh/doc/book/installing/)

curl -o java-16.deb https://download.oracle.com/otn-pub/java/jdk/16.0.1+9/7147401fd7354114ac51ef3e1328291f/jdk-16.0.1_linux-x64_bin.deb?AuthParam=1619231816_fc0bec521b6021aab3fbd967ec132986



dpkg -i java-16.deb 



1. export JAVA_HOME=/usr/lib/jvm/jdk-13.0.1
2. export PATH=$PATH:$JAVA_HOME/bin
3. export CLASSPATH=$JAVA_HOME/lib



```
docker run -p 8080:8080 -p 50000:50000 jenkins

docker run -p 8080:8080 -p 50000:50000 -v /your/home:/var/jenkins_home jenkins

docker run --name myjenkins -p 8080:8080 -p 50000:50000 -v /var/jenkins_home jenkins

docker run --name myjenkins -p 8080:8080 -p 50000:50000 --env JAVA_OPTS=-Dhudson.footerURL=http://mycompany.com jenkins



docker run  -u root  --rm  -d  -p 8080:8080 -p 50000:50000  -v ${PWD}/jenkins-data:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean 

//初始化密码
${PWD}/jenkins-data:/var/jenkins_home/secrets/initialAdminPassword
```



插件里管理安全配置

http://ubuntu:8080/configureSecurity

授权

http://ubuntu:8080/role-strategy





```groovy
pipeline {
    agent any 
    stages {
        stage('Build') { 
            steps {
                // 
            }
        }
        stage('Test') { 
            steps {
                // 
            }
        }
        stage('Deploy') { 
            steps {
                // 
            }
        }
    }
}



node {  
    stage('Build') { 
        // 
    }
    stage('Test') { 
        // 
    }
    stage('Deploy') { 
        // 
    }
}

pipeline { 
    agent any 
    options {
        skipStagesAfterUnstable()
    }
    stages {
        stage('Build') { 
            steps { 
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
}

```