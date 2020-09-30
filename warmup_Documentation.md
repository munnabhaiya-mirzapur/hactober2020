# Warmup Linux Project
# Case : Create a single tier architect environment for php application on AWS instance.

#Server Configure Details:

#AWS Instance Type: t2.micro
#AMI or Server OS: CentOS 7 (x86_64) - with Updates HVM
#Disk Layout: 

#	-- Disk1: 8GB for / (for the root volume)
#	-- Attach 1 more encrypted Volume (Disk 2) with disk accidental termination policy at least 4GB Size of volume when you launched your instance 
#	-- Attach one Elastic IP to your instance 
#	-- Now wait for AWS running status and logged into server using your   keys
#+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

# Prerequisite:

# Check all things like server version, disk size of root volume & extra disk, RAM, CPU….
cat     /etc/os-release 
free   -h
df   -h

#2.	Update server packages
yum    update    -y

#Format disk and mount

#create physical volume
pvcreate	/dev/xvdf
#create volume group
vgcreate	mystorage	/dev/xvdf
#create lvm
lvcreate    --name    mylvm1    --size    800M    mystorage
#formatting disk
mkfs.xfs	/dev/mystorage/mylvm1
mkdir		/home2
#mounting disk
mount    -o    noexec    /dev/mystorage/mylvm1    /home2
#entry in fstab
echo 'UUID=034239ed-33a9-4240-b47c-e80d64bcebla   /home2   xfs   defaults,noexec    0 0' | cat >> /etc/fstab

#Disable Selinux
sed   -i   's/enforcing/disabled/g'   /etc/selinux/config
setenforce 0

#Set timezone of Server to IST
timedatectl	set-timezone	Asia/Kolkata

#Install basic utility like netstat, wget, vim, git, etc.
yum	install	vi	    -y
yum	install	git	    -y
yum	install	net-tools     -y
yum	install	wget	-y


#Configure your motd file on server so that we can get more information dynamically about server whenever we login.
cat <<EOF > /etc/motd.sh
#!/bin/sh
clear
echo "###############################################################
#                 Authorized access only!                     # 
# Disconnect IMMEDIATELY if you are not an authorized user!!! #
#         All actions Will be monitored and recorded          #
###############################################################"
                                                                                                                                                  
echo "+++++++++++++++++++++++++++++++++++++SERVER INFO+++++++++++++++++++++++++++++++++++++++++"                                                     
                                                                                                                                                     
cpu_info=$(cat /proc/cpuinfo | grep -w 'model name' | awk -F: '{print $2}')                                                                          
mem_info=$(cat /proc/meminfo | grep -w 'MemTotal' | awk -F: '{print $2/1024 "M"}')                                                                   
mem_free=$(cat /proc/meminfo | grep -w 'MemFree' | awk -F: '{print $2/1024 "M"}')                                                                    
swap_total=$(cat /proc/meminfo | grep -w 'SwapTotal' | awk -F: '{print $2/1024 "M"}')                                                                
swap_free=$(cat /proc/meminfo | grep -w 'SwapFree' | awk -F: '{print $2/1024 "M"}')                                                                  
disk_total=$(df -h --output=size | head -n 2 | tail -n 1)                                                                                            
disk_free=$(df -h --output=avail | head -n 2 | tail -n 1)                                                                                            
cpu_load=$(cat /proc/loadavg | head -c 15)                                                                                                           
distro=$(cat /etc/*-release | head -n 1)                                                                                                             
public_ip=$(curl ifconfig.co)                                                                                                                        
private_ip=$(hostname -i)                                                                                                                            
                                                                                                                                                     
echo "    CPU: $cpu_info"                                                                                                                            
echo "    Memory: $mem_info"                                                                                                                         
echo "    Swap: $swap_total"                                                                                                                         
echo "    Disk: $disk_total"                                                                                                                         
echo "    Distro: $distro"                                                                                                                           
echo "    CPU Load: $cpu_load"                                                                                                                       
echo "    Free Memory: $mem_free"                                                                                                                    
echo "    Free Swap: $swap_free"                                                                                                                     
echo "    Free Disk: $disk_free"                                                                                                                     
echo "    Public Address: $public_ip"                                                                                                                
echo "    Private Address: $private_ip"                                                                                                              
echo "+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++"                                                   
                                                                                                                                                     
EOF
#make script executable
chmod +x /etc/motd.sh

#Append this script to /etc/profile in order to be executed as last command once a user login.
echo "/etc/motd.sh" >> /etc/profile

#Reboot your server to verify and test all things.
reboot 


#Now start server setup, configuration, test and deploy

#Packages:

#Install Apache 2.4.x server with SSL and proxy module support
yum	install	httpd httpd-tools -y
#rpm -ql httpd : shows the installed packages and directories



systemctl	start	httpd.service
systemctl	enable	httpd.service
yum	install	mod_ssl

#Install PHP v7.x with essential php modules like php-mysql, php-devel, php-mbstring etc which is required by Application.
yum install centos-release-scl   -y
yum     install     rh-php72-php     rh-php72-php-common     rh-php72-php-cli   rh-php72-php-devel     rh-php72-php-gd     rh-php72-php-json     rh-php72-php-mbstring     rh-php72-php-mysqlnd     rh-php72-php-opcache     rh-php72-php-zip sclo-php72-php-mcrypt     rh-php72-php-xml     rh-php72-php-intl

#Symlink the PHP 7.2 Apache modules into place:
ln -s /opt/rh/httpd24/root/etc/httpd/conf.d/rh-php72-php.conf /etc/httpd/conf.d/
ln -s /opt/rh/httpd24/root/etc/httpd/conf.modules.d/15-rh-php72-php.conf /etc/httpd/conf.modules.d/
ln -s /opt/rh/httpd24/root/etc/httpd/modules/librh-php72-php7.so /etc/httpd/modules/
ln -s /opt/rh/rh-php72/root/usr/bin/php /usr/bin/php

#User Setup:

#Create one user blu with passwordless login and blu user home directory should be /home2
mkdir /home2
useradd	-b /home2 blu
#Create public_html folder under user blu home directory
mkdir /home2/blu/public_html
mkdir -m 700 /home2/blu/.ssh
chown blu:blu /home2/blu/.ssh -R

#generate key
ssh-keygen -C "Key generated for php applications" -N "" -f /home2/blu/.ssh/blu-key
cat /home2/blu/.ssh/blu-key.pub  >>  /home2/blu/.ssh/authorized_keys

#fix permissions
chmod 600 /home2/blu/.ssh/authorized_keys
chown blu:blu /home2/blu/.ssh/authorized_keys

# setup user permissions
usermod -a -G apache blu
chown blu:apache /home2/blu/public_html -R
chmod 711 /home2/blu
chmod 2771 /home2/blu/public_html

# VirtualHost Configuration:
cat <<EOF > /etc/httpd/conf.d/blu-php.conf
<VirtualHost *:80>
ServerName bluboy.adhocnw.com
DocumentRoot /home2/blu/public_html
<Directory /home2/blu/public_html/>
Require all granted
DirectoryIndex phpinfo.php
</Directory>
ErrorLog /home2/blu/error.log
</VirtualHost>
EOF

#setup phpinfo page
echo '<?php phpinfo(); ?>' | cat > /home2/blu/public_html/phpinfo.php

#update
yum update -y

#reboot
reboot


























