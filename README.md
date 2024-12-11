# JENKINS
- [INSTALLING JENKINS ON ROCKY 9](#install-jenkins-on-rocky-9)
- [CONFIGURING SSL FOR JENKINS](#configuring-ssl-for-jenkins)
- [ERROR IN ROCKY 9 LINUX REPOS](#error-in-rocky-9-linux-repos)
- [SETTING STATIC IP](#setting-static-ip)
- [ENABLING SSL FOR JENKINS AND CONFIGURING HTTPS](#enabling-ssl-for-jenkins-and-configuring-https)
#### INSTALLING JENKINS ON ROCKY 9
I followed this video to install jenkins on rocky 9;
```
https://www.youtube.com/watch?v=2-L0WohfsqY
```

#### CONFIGURING SSL FOR JENKINS
We will follow that guide;
```
https://medium.com/@touseef.idc/jenkins-installation-and-ssl-certificate-configuration-on-amazon-linux-2023-7394216e5238
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
#### SETTING STATIC IP
I set static IP for my vm, in order to connect same ip to connect my jenkins through my host browser. 
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





