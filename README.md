# Devops Tooling Website Solution

We will implement a tooling website solution which makes access to DevOps tools within the corporate infrastructure easily accessible.

In this project we will implement a solution that consists of following components:

1. **Infrastructure**: AWS
1. Webserver Linux: Red Hat Enterprise Linux 8
1. **Database Server**: Ubuntu 20.04 + MySQL
1. **Storage Server**: Red Hat Enterprise Linux 8 + NFS Server
1. **Programming Language**: PHP
1. **Code Repository**: GitHub

##  Prepare NFS Server

1. Lunch 3 redhat EC2 instances in AWS. Label one `NFS Server` and the other 2 `webserver-1`and `webserver-2`.
1. We create and attach a 15gb volume to our `NFS server` making sure the volume is in the same `availability zone` as our `NFS Server`
1. Login to the `NFS server` and run `lsblk` command to see our newly attached device listed as `xvdf`.
![](https://user-images.githubusercontent.com/18899718/120393891-4f1e5d00-c2f8-11eb-8540-44dfe5e477d3.png)
1. Next, we create a single partition on each of our newly created disk.
1. Run `sudo gdisk /dev/xvdf`
![](https://user-images.githubusercontent.com/18899718/120394370-ed122780-c2f8-11eb-941c-52f3236d722f.png)

 1. Next, we install `lvm2` using `sudo yum install lvm2`, with this we can create a logical volume.
 1.  We create a physical volume using
`sudo pvcreate /dev/xvdf1`
1. Run `sudo pvs`
1. Next, we create a `volume group` named `webdata-vg` by running 
```
sudo vgcreate webdata-vg /dev/xvdf1
```
10. Verify volume group is successfully created by running `sudo vgs`
11.  We use `lvcreate` utility to create 2 logical volumes namely `apps-lv`(which would be used to store data for the website) and `logs-lv` (this will be used to store data for logs). We divide our physical volume between them withe each getting equal halves. 
``` 
sudo lvcreate -n lv-apps -L 5G webdata-vg
sudo lvcreate -n lv-logs -L 5G webdata-vg
sudo lvcreate -n lv-opt -L 100%FREE webdata-vg
```
12. Run `sudo lvs` to verify that the Logical Volume has been successfully created.
![](https://user-images.githubusercontent.com/18899718/120395374-7f66fb00-c2fa-11eb-8642-4931adb36fa8.png)

1. We use `mkfs.xfs` to format the logical volumes with ext4 filesystem
```
sudo mkfs.xfs /dev/webdata-vg/lv-apps
sudo mkfs.xfs /dev/webdata-vg/lv-logs
sudo mkfs.xfs /dev/webdata-vg/lv-opt
```
![](https://user-images.githubusercontent.com/18899718/120396870-f00f1700-c2fc-11eb-8707-7a4f7c2539d7.png)

14. Create mount points on /mnt directory for the logical volumes as follow: 
- Mount `lv-apps` on `/mnt/apps` - To be used by webservers 
- Mount `lv-logs` on `/mnt/logs` - To be used by webserver logs 
- Mount `lv-opt` on `/mnt/opt` - To be used by Jenkins server .

15. create /mnt directory and the necessary folders
``` 
sudo mkdir /mnt/apps
sudo mkdir /mnt/logs
sudo mkdir /mnt/opt
```
16. Mount the logical volumes on their respective diretories.
```
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```
Run `df -h` so the our mounted directories.
![](https://user-images.githubusercontent.com/18899718/120397783-914a9d00-c2fe-11eb-9134-023ba30bd46f.png)

17. We make the our mount persist by updating `/etc/fstab/` with our LV `UUID` 
18.  Run `sudo blkid` to get `UUID`number
19. Run `sudo vi /etc/fstab` and update as shown in the image below
![](https://user-images.githubusercontent.com/18899718/120398295-8c3a1d80-c2ff-11eb-9d45-5ac98a7eaee9.png)
20. Confirm the by running `sudo mount - a`
21. Reload the daemon by running `sudo systemctl daemon-reload`

## Configure NFS Server
1. Install the following:
```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
```
2. Set up permission that will allow our Web servers to read, write and execute files on NFS Server:
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
```
Restart the nfs service
```
sudo systemctl restart nfs-server.service
```
3. Configure access to NFS for clients within the same subnet
```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
```
![](https://user-images.githubusercontent.com/18899718/120398810-81cc5380-c300-11eb-9cd3-6d0a82c8d1bf.png)

4.
```
sudo exportfs -arv
 ```
![](https://user-images.githubusercontent.com/18899718/120399032-e1c2fa00-c300-11eb-8ac6-fd18ce14d69d.png)

5. Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
```
rpcinfo -p | grep nfs
```
![](https://user-images.githubusercontent.com/18899718/120399162-18991000-c301-11eb-9680-9ed0174e49dc.png)

6. > In order for NFS server to be accessible from our webservers, we must open following ports: TCP 111, UDP 111, TCP 2049 and UDP 2049


## Configure Database Server
1. Spin up an Ubuntu instance on aws
2. Install mysql server `sudo apt install mysql-server`
3. configure mysql by running `sudo mysql_secure_installation`
4. Follow the on screen prompts to set up password and login into mysql 
5. log into my mysql using `sudo mysql`
6. create a database named tooling
```
create database tooling;
```
7. create user with name webacces
```
CREATE USER 'webaccess'@'%' IDENTIFIED WITH mysql_native_password BY 'password';

GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'%' WITH GRANT OPTION;
```
8. Exit
9. Edit bind address in mysql configuration file to allow traffic. 
```
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
10. Change bind address to `0.0.0.0`
11. Restart msyql service 
```
sudo systemctl restart mysql
```

## Setup Webservers
> To store shared files for our Web Servers, we will utilize NFS and mount our previously created Logical Volume `lv-apps` to the folder where Apache stores files to be served to the users `/var/www`

### steps to setup webservers
- Configure NFS client (this step must be done on all three servers)
- Deploy a Tooling application to our Web Servers into a shared NFS folder
- Configure the Web Servers to work with a single MySQL database

1. Launch a new EC2 instance with RHEL 8 Operating System
1. Install NFS client
```
sudo yum install nfs-utils nfs4-acl-tools -y
```
3. Mount `/var/www` and target NFS's export `/mnt/apps`
```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
```
4. Verify by running `df -h`
![](https://user-images.githubusercontent.com/18899718/120674736-041c5b00-c45a-11eb-8b13-49e21adcc489.png)
5. Make sure that the changes will persist on Web Server after reboot
```
sudo vi /etc/fstab
```
add the following line
```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```
6. Install apache 
```
sudo yum install httpd -y
```
7. Start apache
```
sudo systemctl enable httpd
sudo systemctl start httpd
```
8. Install PHP and it's dependencies
```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install yum-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module list php
sudo yum module reset php
sudo yum module enable php:remi-7.4
sudo yum install php php-opcache php-gd php-curl php-mysqlnd
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
setsebool -P httpd_execmem 1
```
9. Backup `/var/log/httpd`
```
sudo mv /var/log/httpd /var/log/httpd.bak
```
>It's very important to backup our httpd folder because mounting the httpd folder on NFS server would delete everything inside the folder which would cause serious errors.
10. Mount `/var/log/httpd` and target NFS's export `/mnt/logs`
```
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd
```
![](https://user-images.githubusercontent.com/18899718/120685539-8e1df100-c465-11eb-9d1a-33b4cea24b69.png)
11. Make sure that the changes will persist on Web Server after reboot
```
sudo vi /etc/fstab
```
add the following line
```
<NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd nfs defaults 0 0
```
12. Make sudo to restart deamon by doing 
```
sudo systemctl daemon-reloadn
```
14. 
Copy the contents of apache backup folder into the active apache folder
```
sudo cp -R var/log/httpd.bak/. /var/log/httpd
```
14. Restart apache
```
sudo systemctl restart httpd
```
15. Install git 
```
sudo dnf install git 
```
16. Clone tooling repo
```
git clone https://github.com/realayo/tooling.git
```
17. Change directory to tooling folder and copy html folder to `/var/www`
```
cd tooling
cp - R html /var/www/
```
18. Edit functions.php in html folder and add our database values in there
```
cd /var/www/html
sudo vi functions.php
```
![](https://user-images.githubusercontent.com/18899718/120685663-b3126400-c465-11eb-8086-8da0718c0e77.png)
Save and exit
19. Repeat steps 1-19 on the other 2 webservers

### Install mysql on webserver
1. Install mysql 
```
sudo yum install mysql-server -y
```
2. Navigate into tooling folder and apply tooling script
```
cd tooling
mysql -h 172.31.45.212 -u webaccess -p tooling < tooling-db.sql
```
3. On Database server
create a user named `myuser` with password `password`
```
CREATE USER 'myuser'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
GRANT ALL PRIVILEGES ON tooling.* TO 'myuser'@'%' WITH GRANT OPTION;
```
4. Insert new records into table `users`
```
INSERT INTO users (id, username, password, email, user_type, status) VALUES ('1', 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
```
5. Repeat steps 1-4 on the other 2 web servers

Finally, navigate to `http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php` on your browswer and make sure you can login into the website with `myuser` user.
![](https://user-images.githubusercontent.com/18899718/120688446-b0653e00-c468-11eb-81c9-48b0d71d0c8f.png)

![](https://user-images.githubusercontent.com/18899718/120715303-0a2a3000-c48a-11eb-888d-18e39d2ce188.png)