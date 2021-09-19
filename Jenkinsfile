node{
    def app
    def product="cdac-project"
    stage('clean workspace'){
        echo 'Clean Workspace '
        cleanWs()
    }
    stage('Clone repository') {
        echo "Cloning git repository to workspace"
        checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'cdac_github_token', url: "https://github.com/akssharma1994/${product}.git"]]])
    }

    stage('Build image') {
        echo 'Build the docker flask image'
        app = docker.build("8077103273/${product}")
    }

    stage('Test image') {
        echo 'Test the docker flask image'
        app.inside {
            sh 'python test.py'
        }
    }
    stage('Push image') {
        echo 'Push image to the docker hub'
        docker.withRegistry('https://registry.hub.docker.com', 'docker_cred') {
            app.push("${env.BUILD_NUMBER}")
            app.push("latest")
        }
    }
    stage('Update the deployment file'){
    echo 'update the deployment files to re-apply it on deployment'

     sh "sed -i s/%IMAGE_NO%/${env.BUILD_NUMBER}/g flask-deployment.yaml"
     sh "cat flask-deployment.yaml"
    }
    stage('Deploy the flask app'){
      echo 'Deploy the flasj image at AWS EKS, on Cluster already present in EKS'
      withCredentials([[
              $class: 'AmazonWebServicesCredentialsBinding',
              credentialsId: 'AWS_CREDS',
              accessKeyVariable: 'AWS_ACCESS_KEY_ID',
              secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
          ]]){
      sh """

              
              kubectl version --short --client
              eksctl version
              aws eks --region us-east-1 update-kubeconfig --name ${product}-cluster
              kubectl get svc
              echo "Execute the deployment"
              kubectl create namespace ${product}
              if [ $? -eq 0 ]; then
                  echo "namespace ${product} already exists"
                  kubectl get all -n ${product}
              else
                  echo "create ${product} namespace"
                  kubectl create namespace ${product}
              fi
              echo "Apply the deployment"
              kubectl apply -f flask-deployment.yaml
              echo "Create the flask service"
              kubectl apply -f flask-service.yaml
              sleep 5s
              echo "\n\n Deployment details \n\n"
              kubectl get all -n ${product}

              echo "Deployment done successfully"
        """
    }  }
    stage('Deployment Test'){
        echo 'Test the deployment using curl on service external address'
        withCredentials([[
                $class: 'AmazonWebServicesCredentialsBinding',
                credentialsId: 'AWS_CREDS',
                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
            ]]){
        sh """
                echo $PATH
                kubectl get all -n ${product}
                sleep 20s
                EXTERNAL_IP=`kubectl get service flask-service -n ${product} | awk 'NR==2 {print $4}'`
                STATUS_CODE=`curl -s -o /dev/null -w "%{http_code}" http://${EXTERNAL_IP}:5000`
                echo $STATUS_CODE
                if [ $STATUS_CODE -eq 200 ]; then
                    echo "Deployment done successfully"
                else
                    echo "\n\nApplication not responding deployment Failed\n\n "
                    exit 1
                fi
          """
        } }
        stage('Clean docker images from local') {
      sh """
          sudo docker images -a | grep "${product}" | awk '{print $3}' | xargs docker rmi -f
      """

  }
}
