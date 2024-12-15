# JENKINS
- [INSTALLING JENKINS ON ROCKY 9](#install-jenkins-on-rocky-9)
- [ERROR IN ROCKY 9 LINUX REPOS](#error-in-rocky-9-linux-repos)
- [NETWORK CONFIGURATION](#network-configuration)
- [ENABLING SSL FOR JENKINS AND CONFIGURING HTTPS](#enabling-ssl-for-jenkins-and-configuring-https)
- [CREATING DECLARTIVE PIPELINE WITH GIT REPO](#creating-declartive-pipeline-with-git-repo)
- [ANSWERS FOR ADDITIONAL QUESTIONS](#answers-for-additional-questions)
#### INSTALLING JENKINS ON ROCKY 9
I followed this video to install jenkins on rocky 9;
```
https://www.youtube.com/watch?v=2-L0WohfsqY
```

#### ERROR IN ROCKY 9 LINUX REPOS
When we do `dnf update`, Error is thrown;
```
 Failed to download metadata for repo 'baseos': Yum repo downloading error: Downloading error(s): repodata/230cec9b-e893-4f8d-aedf-11663dcd15e8-PRIMARY.xml.gz - Cannot download, all mirrors were already tried without success; repodata/f52410cf-c203-4321-ad32-728433857cf5-FILELISTS.xml.gz - Cannot download, all mirrors were already tried without success; repodata/f52410cf-c203-4321-ad32-728433857cf5-GROUPS.xml.gz - Cannot download, all mirrors were already tried without success
```
This shows related to mirrors for baseos, we can solve that error by configuring the `/etc/yum.repos.d/rocky.repo` by commented the mirrorlist variable and uncommented baseurl variable under the `[baseos]`;
```
[baseos]
name=Rocky Linux $releasever - BaseOS
mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever$rltype
baseurl=http://dl.rockylinux.org/$contentdir/$releasever/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
countme=1
metadata_expire=6h
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
```
And also we should do the same for `[appstream]` because, same error is thrown;
```
[appstream]
name=Rocky Linux $releasever - AppStream
#mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=AppStream-$releasever$rltype
baseurl=http://dl.rockylinux.org/$contentdir/$releasever/AppStream/$basearch/os/
gpgcheck=1
enabled=1
countme=1
metadata_expire=6h
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
```
#### NETWORK CONFIGURATION
Set up 2 network adapter;  `HOST-ONLY ADAPTER` is for internal communication like my host pc and vm, `NAT` is for external communication internet and vm. 

#### ENABLING SSL FOR JENKINS AND CONFIGURING HTTPS
1. __Install Nginx__

Nginx will act as a reverse proxy for Jenkins to make it accessible via HTTP/HTTPS:
```
sudo dnf install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```
2. __Configure Nginx to Proxy Jenkins__
Create an Nginx configuration file for Jenkins:

```
sudo nano /etc/nginx/conf.d/jenkins.conf
```
Add the following configuration to proxy requests to Jenkins running on port 8080:
```
upstream jenkins {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name 192.168.56.101;  

    location / {
        proxy_pass http://jenkins;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
Test and restart Nginx:
```
sudo nginx -t
sudo systemctl restart nginx
```
Now Jenkins should be accessible at http://192.168.56.101.
3. __Handling Self-Signed SSL Certificates__
If you're testing locally and don't have a domain name, you can generate a self-signed SSL certificate. Here's how:

Generate Self-Signed Certificate:
```
sudo mkdir -p /etc/ssl/private
sudo openssl req -x509 -newkey rsa:4096 -keyout /etc/ssl/private/jenkins.key -out /etc/ssl/certs/jenkins.crt -days 365 -nodes
```
Configure Nginx with Self-Signed Certificate:

- Modify your Nginx configuration to point to the self-signed certificate:
```
server {
    listen 443 ssl;
    server_name 192.168.56.101;

    ssl_certificate /etc/ssl/certs/jenkins.crt;
    ssl_certificate_key /etc/ssl/private/jenkins.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
- Testing SSL: If SSL isn't working, test it by using:
```
sudo nginx -t
sudo systemctl restart nginx
```
After completing the above steps, we should be able to access Jenkins securely over HTTPS using self-signed certificate. Open browser and go to `https://192.168.56.101`. We see a security warning , but we can bypass it for testing purposes.

#### CREATING DECLARTIVE PIPELINE
1. We need add new item then change the definition of the pipeline as `pipeline script from SCM`, and then give the url of the project `(https://github.com/BilgeKaanGencdogan/learn-jenkins-app)` and write script path for jenkins `Jenkinsfile` then click save the pipeline
2. We should create new file with same name as the script path for jenkins in previous step.
3. And finally write the pipeline;
```Jenkins
pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'f64217ac-7042-4c89-a7c2-727934a56cab'
        NETLIFY_AUTH_TOKEN = credentials('token-netlify')
    }

    stages {
        stage('Install Retire.js') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '--user root'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Install retire.js globally
                    npm install -g retire
                    retire
                '''
            }
        }

        // stage('Install Bearer CLI') {
        //     agent {
        //         docker {
        //             image 'node:18-alpine'
        //             args '--user root'
        //             reuseNode true
        //         }
        //     }
        //     steps {
        //         sh '''
        //              apk add --no-cache curl git
        //             curl -sfL https://raw.githubusercontent.com/Bearer/bearer/main/contrib/install.sh | sh
        //             ls -la ./bin
        //             ./bin/bearer scan .
        //         '''
        //     }
        // }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '--user root'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    whoami
                    ls -la
                    node --version
                    npm --version
                    npm ci --unsafe-perm
                    npm run build
                    ls -la
                '''
            }
        }

        stage('OWASP Dependency-Check Vulnerabilities') {
            steps {
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'OWASP Dependency-Check Vulnerabilities'
                
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            args '--user root'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm run test:ci
                        '''
                    }

                    post {
                        always {
                            junit 'test-results/junit.xml'
                        }
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    args '--user root'
                    reuseNode true
                }
            }
            steps {
                    // sh '''
                    //  # Install netlify-cli locally in the project
                    // npm install netlify-cli --unsafe-perm

                    // # Verify installation
                    // node_modules/.bin/netlify --version

                    // # Deploy using netlify-cli
                    // echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    // netlify login
                    // node_modules/.bin/netlify status
                    // node_modules/.bin/netlify deploy --dir=build --prod
                    // '''

                sh '''
                    echo "Deploying...."
                    '''
            }
        }
    }

    // post {
    //     always {
    //         script {
    //             // Ensure the directory exists and has proper permissions
    //             sh '''
    //                 if [ -d "playwright-report" ]; then
    //                     echo "Playwright report directory exists."
    //                 else
    //                     echo "Playwright report directory does not exist. Creating it."
    //                     mkdir -p playwright-report
    //                 fi

    //                 chmod -R 777 /var/lib/jenkins/workspace/learn-jenkins-app/playwright-report
                    
    //             '''
    //         }

    //         // Publish Playwright report
    //         publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
    //     }
    // }

     post 
     {
     always {
        script {
            if (currentBuild.result == 'FAILURE') {
                // Logic to create a GitHub issue
                sh """
                curl -X POST -H "Authorization: token ${env.GITHUB_TOKEN}" \
                     -H "Accept: application/vnd.github+json" \
                     https://api.github.com/repos/BilgeKaanGencdogan/learn-jenkins-app/issues \
                     -d '{
                            "title": "Build failed",
                            "body": "The build failed with the following error: ${currentBuild.result}",
                            "labels": ["build-failure"]
                         }'
                """
            }
        }
    }
}

}

```
There are some lines commented because whatever I do, I could not resolve the issues when they are added, most of the errors are related to permission issues Even though I've configured permissions, but they are thrown anyway. I am going to provide one of them below to give an idea;
1. even though my current user is root, this error kept thrown. I was trying to deploy the application to the netlify.
 ```
+ npm install '--unsafe-perm=true' netlify-cli
npm error code EACCES
npm error syscall mkdir
npm error path /var/lib/jenkins/workspace/learn-jenkins-app/node_modules/playwright/node_modules/fsevents
npm error errno -13
npm error [Error: EACCES: permission denied, mkdir '/var/lib/jenkins/workspace/learn-jenkins-app/node_modules/playwright/node_modules/fsevents'] {
npm error   errno: -13,
npm error   code: 'EACCES',
npm error   syscall: 'mkdir',
npm error   path: '/var/lib/jenkins/workspace/learn-jenkins-app/node_modules/playwright/node_modules/fsevents'
npm error }
npm error
npm error The operation was rejected by your operating system.
npm error It is likely you do not have the permissions to access this file as the current user
npm error
npm error If you believe this might be a permissions issue, please double-check the
npm error permissions of the file and its containing directories, or try running
npm error the command again as root/Administrator.
npm error A complete log of this run can be found in: /var/lib/jenkins/workspace/learn-jenkins-app/${npm_config_cache}/_logs/2024-12-14T17_06_49_438Z-debug-0.log
script returned exit code 243
```
In IDE, Every command is working fine, I have done it in the IDE before writing but somehow they did not work in the pipeline. 
#### ANSWERS FOR ADDITIONAL QUESTIONS
1. __Could there be alternative methods to create this Jenkins pipeline other than the one you used?__
   I've used declaretive pipeline, and it feels easy for me regardles of the errors I've gotten.
   There is one more which is scripted pipeline which is the original pipeline syntax for Jenkins, it uses Groovy scripting language to define pipelines. It provides a lot of flexibility and control over the workflow, but can be more complex and verbose.

2.  __If the code you had to compile required the use of another OS (e.g., macOS, Windows), how would you compile that code?__
   I would use docker. Most probably there are `Docker images` mimicking the required environment (e.g., Windows Containers for Windows code, Linux Containers for Linux code).

3. __Is it possible to run the Jenkins pipeline automatically every time the source code is modified? How?__
   There is webhook which does that. Configuration pf  source control system (e.g., GitHub, GitLab, Bitbucket) to send a webhook to your Jenkins server whenever changes are pushed to the repository.

4. __How would you set up a highly available Jenkins installation?__
-Master-Slave Architecture: Use a Jenkins master for managing jobs and multiple agent nodes for builds.
-Active-Passive Setup: Deploy two Jenkins masters (active and standby) with a load balancer to switch traffic if the active master fails.
-Shared Storage: Use NFS or cloud storage for job configurations and artifacts.
-External Database: Store Jenkins metadata in an external database like MySQL for easy recovery.
-Kubernetes: Deploy Jenkins in Kubernetes for auto-scaling and fault tolerance.
-Backups: Regularly back up Jenkins configurations and data for disaster recovery.






