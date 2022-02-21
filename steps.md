# DEVOPS TOOLING WEBSITE SOLUTION


##### STEP 1 – PREPARE NFS SERVER

Spin up a new EC2 instance with RHEL Linux 8 Operating System.

Based on your LVM experience from Project 6, Configure LVM on the Server.

Instead of formating the disks as ext4 you will have to format them as xfs

Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs
img

Create mount points on /mnt directory for the logical volumes as follow:
Mount lv-apps on /mnt/apps – To be used by webservers
Mount lv-logs on  /mnt/logs – To be used by webserver logs
Mount lv-opt on  /mnt/opt – To be used by Jenkins server in Project 8
img

Install NFS server, configure it to start on reboot and make sure it is up and running
*sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service*
img

- Set up permission that will allow our Web servers to read, write and execute files on NFS:
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service

- Configure access to NFS for clients within the same subnet
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
  img
- Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
*rpcinfo -p | grep nfs*
  img
  
 -  In order for NFS server to be accessible from your client, we must also open following ports: TCP 111, UDP 111, UDP 2049
  img
  
  
 ##### STEP 2 — CONFIGURE THE DATABASE SERVER
  - Install MySQL server
  - Create a database and name it tooling
  - Create a database user and name it webaccess
- Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr
  img
  
  ##### Step 3 — Prepare the Web Servers
  During the next steps we will do following:

1. Configure NFS client (this step must be done on all three servers)
2. Deploy a Tooling application to our Web Servers into a shared NFS folder
3. Configure the Web Servers to work with a single MySQL database

  - Launch a new EC2 instance with RHEL 8 Operating System
- Install NFS client
*sudo yum install nfs-utils nfs4-acl-tools -y

  Mount /var/www/ and target the NFS server’s export for apps
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www
Verify that NFS was mounted successfully by running df -h. Make sure that the changes will persist on Web Server after reboot:
img
sudo vi /etc/fstab
add following line

<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
img
  
Install Remi’s repository, Apache and PHP
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

setsebool -P httpd_execmem 1

 ##### Repeat steps 1-5 for another 2 Web Servers.
  
  - Verify that Apache files and directories are available on the Web Server in /var/www and also on the NFS server in /mnt/apps. 
  img
  img
  
- Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. 
  
- Fork the tooling source code from Darey.io Github Account to your Github account. 
  
- Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html
  cp -R /html/. /var/www/html
  - open TCP port 80 on the Web Server.
  - disable SELinux sudo setenforce 0
- To make this change permanent – open following config file sudo vi /etc/sysconfig/selinux and set SELINUX=disabledthen restrt httpd.
 - Update the website’s configuration to connect to the database (in /var/www/html/functions.php file). 
  img
 - Apply tooling-db.sql script to your database using this command 
  mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql
img

  
