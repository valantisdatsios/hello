pipeline {
    agent any
    
    environment {
        hostIp = "10.78.60.254"
        hostUser = "root"
        containerName = "vanilla_container"
        containerRunCmd = "docker run --init --rm -d --name ${containerName} \
                        -v ${containerName}_vol_etc_rundeck:/etc/rundeck \
                        -v ${containerName}_vol_var_rundeck:/var/rundeck \
                        -v ${containerName}_vol_var_lib_rundeck:/var/lib/rundeck \
                        -v ${containerName}_vol_var_log_rundeck:/var/log/rundeck \
                        -v ${containerName}_vol_var_lib_rundeck_logs:/var/lib/rundeck/logs \
                        -v ${containerName}_vol_var_lib_rundeck_storage:/var/lib/rundeck/var/storage \
                        -v ${containerName}_vol_rundeck_plugins:/opt/rundeck-plugins \
                        -v ${containerName}_vol_var_lib_mysql:/var/lib/mysql \
                        -v ${containerName}_vol_opt_ncs:/opt/ncs \
                        -p 4440:4440 \
                        -e EXTERNAL_SERVER_URL=http://${hostIp}:4440 \
                        -e NO_LOCAL_MYSQL=false \
                        -e RUNDECK_STORAGE_PROVIDER=db \
                        -e RUNDECK_PROJECT_STORAGE_TYPE=db \
                        -e DATABASE_ADMIN_USER=root \
                        -e DATABASE_ADMIN_PASSWORD=123456 \
                        -e RUNDECK_PASSWORD=123456 \
                        -t vanilla:latest"
        containerExecCmd = "docker exec -i ${containerName} echo hello"
        containerStopCmd = "docker stop ${containerName}"
        createUtilsProjectCmd = "docker exec -i vanilla_container bash -c 'export RD_URL=http://localhost:4440 && export RD_USER=admin && export RD_PASSWORD=admin && rd projects create --project Utils'"
    }
    
    stages {
        stage('Vanilla Directory Preparation') {
            steps {
                // Create vanilla_dir to sync from P4
                sh "mkdir -p vanilla_dir"
                // Change to vanilla_dir
                dir('vanilla_dir'){
                    // Sync contents of ncs_container from P4
                    p4sync charset: 'none', 
                           credential: 'perforce_server', 
                           format: 'jenkins-${NODE_NAME}-${JOB_NAME}-${EXECUTOR_NUMBER}-ncs_container', 
                           populate: autoClean(delete: true, modtime: false, parallel: [enable: false, minbytes: '1024', minfiles: '1', threads: '4'], pin: '', quiet: true, replace: true, tidy: false), 
                           source: depotSource('//depot/CloudOps/trunk/ncs_devel/automation/ncs_container/...')
                }
                // Create ncs directory to sync from P4
                sh "mkdir -p vanilla_dir/content/var/tmp/ncs"
                dir('vanilla_dir/content/var/tmp/ncs') {
                    // Sync services from P4
                    p4sync charset: 'none', 
                           credential: 'perforce_server', 
                           format: 'jenkins-${NODE_NAME}-${JOB_NAME}-${EXECUTOR_NUMBER}-ncs_DyOS', 
                           populate: autoClean(delete: true, modtime: false, parallel: [enable: false, minbytes: '1024', minfiles: '1', threads: '4'], pin: '', quiet: true, replace: true, tidy: false), 
                           source: depotSource('//depot/CloudOps/trunk/ncs_devel/...')
                }
            }
        }
        
        stage('Build Vanilla docker image') {
            steps {
                // Change to vanilla_dir
                dir('vanilla_dir'){
                    script {
                        def vanilla_image = docker.build("vanilla:latest")
                    }
                }
                sh "docker images"
            }
        }
        
        stage('Run Vanilla container') {
            steps {
                sshagent (credentials: ['ssh_credentials_for_254']) {
                    sh 'ssh -o StrictHostKeyChecking=no ${hostUser}@${hostIp} ${containerRunCmd}'
                }
                // Added some 3min sleep time, this should be replaced with supervisorctl status rundeck
                sleep time: 3, unit: 'MINUTES'
            }
        }
        
        stage('Populate DB with Projects/Jobs') {
            steps {
                sshagent (credentials: ['ssh_credentials_for_254']) {
                    sh returnStdout: true, script: 'ssh -o StrictHostKeyChecking=no ${hostUser}@${hostIp} ${createUtilsProjectCmd}'
                }
            }
        }
        
        /*stage('Stop/Remove Vanilla container') {
            steps {
                sshagent (credentials: ['ssh_credentials_for_254']) {
                    sh 'ssh -o StrictHostKeyChecking=no ${hostUser}@${hostIp} ${containerStopCmd}'
                    sh 'ssh -o StrictHostKeyChecking=no ${hostUser}@${hostIp} docker volume prune -f'
                }
            }
        }*/
    }
}
