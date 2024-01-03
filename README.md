
# Implementing Wordpress Web Solution

## Step-1 Prepare Web Server

- Create an Ec2 Instance server using AWS Provider
- Under **EBS**: Elastic Block Store create 3 storage volumes and `attach` to the Instance. This will serve as an additional external storage for our EC2 Instance.
- The EBS should be under the same availability zone.

![1 Ec2_Project6](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/d5639ba8-fd8c-4080-b9d6-b16082ee2ccd)

![12 EBS_Volume](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/6f844661-de18-4596-8e0b-c385478b121b)

- Log into EC2 instance via SSH, view the additional disk that have been attached to the instance by running `lsblk` command. 
  
![2 Show_Of_Device](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/e6b0dbdd-7dbb-44f5-8ab7-c42ad642f7d7)

To see all the mounts and free space on the server run `df -h` 

![13 List_free_space](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/de02a373-ce32-4ee0-a209-f0d1e9c2c126)

- Create partitions on each EBS volume on the server using `gdisk`
- Follow the step using `gdisk` for the remainder 2 other volumes.

![3 LVM_Set_Up](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/1fcdebb6-93b8-4f50-abb4-7e464661bf9c)

- Use `lsblk` to view the updated partitions

![4 lsblk](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/b6951000-87fe-442f-a6ff-11e44b794260)

- install `lvm` utility in order to create logical volumes on the linux server. Run `sudo yum install lvm2`
  
![5 Sudo_Install_LVM2](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/da4bc9a5-2391-429f-8fdd-34f3d22b490d)

- To verify `LVM2` package has been installed on the server use `which lvm`
  
![6 To_verify_Installation](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/806da2b9-1501-4510-9269-a4337bd47df5)

Next step is to mark the newly created partitions as physical volumes using `sudo pvcreate /dev/<partition>` 

![7 Pv_create](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/37b07626-6298-483f-a16c-bbc2afdc902d)

- `sudo pvs` to view

![8 Sudo_Pvs](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/5d2ca173-3569-48d6-a754-88cab532b526)

- Next step is to group all the physical volumes into a volume group using `sudo vgcreate <group_name> <pv_path1> <pv_path2>`. 
- `sudo vgs` to view the newly created volume group

![9 Create_Volume_Group](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/379341e3-8734-454b-a0f9-04ff1117eadf)

- Create logical volume for the volume group `sudo lvcreate -n <lv_name> -L <lv_size> <vg_name>`. Example `sudo lvcreate -n <apps-lv> -L <14G> <webdata-vg>`
- `sudo lvs` to view logical volume.

![10 View_logical_Volume](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/2f032e11-6a6c-42f0-9ea1-d873d574b26d)

- Our logical volumes are ready to be used as filesystems for storing application and log data.
- Create filesystems for the logical volumes created. 

![11 Filesystem](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/4543d03b-92ba-446c-a715-e331c0c78ff1)

- The apache webserver uses the `HTML` folder in the `var` directory to store its web content. We create this directory and also a directory for collecting log data of our application.

![14 Apache_Content_Library](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/2208e63b-a278-4e78-a971-d9cb3453d6c5)

- For our filesystem to be used by the user we mount it on the apache directory. We mount the log filesystem to the log directory
```
sudo mount /dev/webdata-vg/apps-lv  /var/www/html/
```
- backup the log directory /var/log/. to the /home/recovery/logs before mounting. Mouting a directory that's not empty will remove everything within that directory.
- Run `sudo rsync -av /var/log/. /home/recovery/logs
  ```
  sudo mount /dev/webdata-vg/logs-lv  /var/log/
  ```
  - Restote run `sudo rsync -av /home/recovery/logs /var/log/
 
 ## Persisting Mount Points

 - To ensure that all of our mounts points are not erased after server restart we need to persist the mount point by configurating the `/etc/fstab` directory
 - `sudo blkid`

![15 Blkid](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/569b54e1-78c5-4f45-80ea-01c8320f7b06)

`sudo vi /etc/fstab/`

![16 Persisting_Mount_Point](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/2a12e3e8-5cd2-4976-b48e-16151c5de022)

- Test the mount point by running `sudo mount -a' - if no error then we did everything correctly
- Restart the daemon `sudo systemctl deamoin-reload`

## Prepare The Database Server (MYSQL)

- Repeated all the steps taken to configure the web server on the DB server. Changed the `apps-lv` logical volume to `db-lv`

## Configuring Web Server
- Run updates and install httpd on web server
```
yum install -y update
sudo yum -y install wget httpd php php-mysqlnd php-fpm php-json
```
- For apache to restart automatically up on server reboot : `sudo systemctl enable httpd`
Start web server and check the status

![17 Start_Web_Server](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/75158225-56b7-43d0-8909-0e76f4549253)

```
sudo systemcyl start httpd
sudo systemctl status httpd
```
- Installing php and its dependencies

```
sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
sudo yum install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm -y
sudo yum module reset php -y
sudo yum module enable php:remi-8.0 -y
sudo yum install php php-opcache php-gd php-curl php-mysqlnd -y
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
sudo setsebool -P httpd_execmem 1
```
- Restart the web server: `sudo systemctl reload httpd`

## Downloading wordpress and moving it into the web content directory

```
mkdir wordpress
cd   wordpress
sudo wget http://wordpress.org/latest.tar.gz
sudo tar xzvf latest.tar.gz
sudo rm -rf latest.tar.gz
cp wordpress/wp-config-sample.php wordpress/wp-config.php
cp -R wordpress /var/www/html/
```
- Configure SELinux Policies
  
```  
sudo chown -R apache:apache /var/www/html/wordpress
sudo chcon -t httpd_sys_rw_content_t /var/www/html/wordpress -R
sudo setsebool -P httpd_can_network_connect=1
```
## Installing MySQL on DB Server
```
sudo yum update
sudo yum install mysql-server
```

- To ensure that database server starts automatically on reboot or system startup
```
sudo systemctl restart mysqld
sudo systemctl enable mysqld
```
![18 Starting_DB_Server](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/04ea6a8a-6e97-451d-ac02-795fb80cd4ec)

- Configure Database to work with wordpress

```
sudo mysql
CREATE DATABASE wordpress;
CREATE USER `myuser`@`<Web-Server-Private-IP-Address>` IDENTIFIED BY 'mypass';
GRANT ALL ON wordpress.* TO 'myuser'@'<Web-Server-Private-IP-Address>';
FLUSH PRIVILEGES;
SHOW DATABASES;
exit
```
- Ensure that we add port 3306 on our db server to allow our web server to access the database server.
![19 Security_Group](https://github.com/lucm9/My-Personal-Project-Documentation/assets/96879757/a33ed3bc-479e-4d12-ac93-7581c173a4e2)

## Connecting Web Server to DB Server
Installing mySQl client on the web server so we can connect to the db server
```
sudo yum install mysql
sudo mysql -u admin -p -h <DB-Server-Private-IP-address>
```
