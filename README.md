# JENKINS
- [INSTALLING JENKINS ON ROCKY 9](#install-jenkins-on-rocky-9)
- [CONFIGURING SSL FOR JENKINS](#configuring-ssl-for-jenkins)
- [ERROR IN ROCKY 9 LINUX REPOS](#error-in-rocky-9-linux-repos)
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


