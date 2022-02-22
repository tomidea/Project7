# DEVOPS TOOLING WEBSITE SOLUTION


##### STEP 1 – PREPARE NFS SERVER

Spin up a new EC2 instance with RHEL Linux 8 Operating System.

Based on your LVM experience from Project 6, Configure LVM on the Server.

Instead of formating the disks as ext4 you will have to format them as xfs

Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs
  
 <img width="515" alt="3 lv" src="https://user-images.githubusercontent.com/51254648/155039047-7bcdc984-7945-4f09-99ed-71ea0d8b7125.png">

<img width="631" alt="logical volumes" src="https://user-images.githubusercontent.com/51254648/155039049-4c51976d-5f17-4c25-a5a0-42da02719dd4.png">

Create mount points on /mnt directory for the logical volumes as follow:
Mount lv-apps on /mnt/apps – To be used by webservers
Mount lv-logs on  /mnt/logs – To be used by webserver logs
Mount lv-opt on  /mnt/opt – To be used by Jenkins server in Project 8



Install NFS server, configure it to start on reboot and make sure it is up and running
*sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service*

<img width="642" alt="nfs install" src="https://user-images.githubusercontent.com/51254648/155039055-176f81ec-80e2-45f5-94da-dee63aaebb55.png">


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
  
  <img width="255" alt="export mounts" src="https://user-images.githubusercontent.com/51254648/155039058-c480fc31-b58a-4e02-a5f6-e4e320667e5e.png">

- Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
*rpcinfo -p | grep nfs*
 
  <img width="379" alt="nfs ports" src="https://user-images.githubusercontent.com/51254648/155039060-c1df5a71-1433-4a36-a64e-8e5fab0e70ec.png">

  
 -  In order for NFS server to be accessible from your client, we must also open following ports: TCP 111, UDP 111, UDP 2049
  

  
  
 ##### STEP 2 — CONFIGURE THE DATABASE SERVER
  - Install MySQL server
  - Create a database and name it tooling
  - Create a database user and name it webaccess
- Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr
  
  
  <img width="527" alt="db conf" src="https://user-images.githubusercontent.com/51254648/155039062-f0f8b161-e2b9-4e17-891b-eeb64199b9b2.png">

  
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

  <img width="438" alt="verify nfs mount" src="https://user-images.githubusercontent.com/51254648/155039065-4cb13406-4667-42ba-9ff9-ff5a28ad2761.png">

sudo vi /etc/fstab
add following line

<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0

  <img width="704" alt="add fstab conf" src="https://user-images.githubusercontent.com/51254648/155039068-2fd9468d-38e3-4468-933f-e758dc87d91b.png">

  
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
 
  <img width="316" alt="nfs view" src="https://user-images.githubusercontent.com/51254648/155039069-33a93a98-62ae-4f1d-b43c-83b88fe90479.png">

<img width="362" alt="client view" src="https://user-images.githubusercontent.com/51254648/155039071-593e51d3-b261-46fc-a1a2-74bdf358e482.png">

  
- Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs. 
  
 <img width="709" alt="fstab log conf" src="https://user-images.githubusercontent.com/51254648/155039073-5625bc69-e035-4cce-886d-a5901e49fbf0.png">
  
  
- Fork the tooling source code from Darey.io Github Account to your Github account. 
  
- Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html
 
  *cp -R /html/. /var/www/html*
  
  - open TCP port 80 on the Web Server.
  - disable SELinux sudo setenforce 0
- To make this change permanent – open following config file sudo vi /etc/sysconfig/selinux and set SELINUX=disabledthen restart httpd.
 - Update the website’s configuration to connect to the database (in /var/www/html/functions.php file). 
 
<img width="530" alt="connect to db conf" src="https://user-images.githubusercontent.com/51254648/155039074-9babef94-5822-4e05-bd76-4faabbc21905.png">

 - Apply tooling-db.sql script to your database using this command 
  mysql -h <databse-private-ip> -u <db-username> -p <db-pasword> < tooling-db.sql

                                                                                 
- Create in MySQL a new admin user with username: myuser and password: password:
  INSERT INTO users (id, username, password, email, user_type, status) VALUES (2, "myuser", "password", "user@mail.com", "admin", "1");
  - Open the website in your browser http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php and login with myuser

  <img width="1248" alt="tooling page" src="https://user-images.githubusercontent.com/51254648/155039076-7b40f9a0-28e8-4170-b6d0-54f4fd16e7ec.png">


  
