# TPC-W
TPC-W benchmark

## Installation 

### Requirements
This implementation requires at least two virtual machines: 
- 1 x MySQL Server
- n X Webserver (Tomcat7)

All the virtual machines are based on Ubuntu.

### Mysql Server
Install MySQL server.
```
sudo apt update
sudo apt install mysql-server
```
Create the TPC-W database, so access to the database
```
mysql -u root -p  #Insert the mysql password
```
and run the commands:
```
CREATE DATABASE std;
GRANT ALL PRIVILEGES ON std.* TO tpcw@'%' IDENTIFIED BY "pswd" WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON std.* TO tpcw@'localhost' IDENTIFIED BY "pswd" WITH GRANT OPTION;
FLUSH PRIVILEGES;
quit;
```

If you are deploying the Mysql server on a different server than the web server, you need to open the port to the other servers in this way:
```
nano /etc/mysql/my.cnf
```

and change `bind-address = 127.0.0.1` into
```
bind-address            = 0.0.0.0
```




### Web server(s)

#### Install packages and TPC-W
```
sudo apt update
sudo apt install tomcat7 default-jre default-jdk ant perl libmysql-java libservlet3.0-java build-essential
```
Take a copy of TPC-W
```
cd /opt
sudo mkdir TPC-W
sudo chmod -R 777 TPC-W
git clone https://github.com/dipietro-salvatore/TPC-W.git TPC-W
```
#### Set Tomcat
Set up Tomcat. you need to replace the current Tomcat configuration file:
```
cd TPC-W/tomcat-config
sudo cp /etc/tomcat7/context.xml /etc/tomcat7/context.xml.orig
sudo cp /etc/tomcat7/tomcat-users.xml /etc/tomcat7/tomcat-users.xml.orig
sudo cp /etc/tomcat7/web.xml /etc/tomcat7/web.xml.orig
sudo cp context.xml /etc/tomcat7/context.xml
sudo cp tomcat-users.xml /etc/tomcat7/tomcat-users.xml 
sudo cp web.xml /etc/tomcat7/web.xml
```
Restart Tomcat

```
sudo service tomcat7 restart
```

#### Configure TPC-W

```
cd ../java-tpcw/

nano tpcw.properties 
```
And change the JDBC settings like this:

```
#jdbc.path=jdbc\:mysql\://localhost\:3306/std?user\=tpcw&password\=pswd
jdbc.path=jdbc\:mysql\://10.11.0.6\:3306/std?user\=tpcw&password\=pswd
```
 
Change the num.item and num.eb as you like. For example:
```
num.item=100000
num.eb=100
```

Now we need to set the java packages into the main.property file. 
So first run these two commands and take note of the full name of these two packages.
```
ls /usr/share/java/*servlet*
ls /usr/share/java/*mysql*
```

Now you need to set them into this file  main.properties. For example:
```
nano main.properties
```
and set them:
```
#<!-- Path to servlet.jar, change this ... -->
cpServ=/usr/share/java/servlet-api-3.0.jar

#<!-- Path to the JDBC driver for your DBMS, change this ... -->
cpJDBC=/usr/share/java/mysql.jar

#<!-- Directory where tpcw.war will be put with task 'inst' -->
webappDir=/var/lib/tomcat7/webapps
```

#### Compile TPC-W
Compile the application. Run on all the webservers:
```
ant dist        #Create a .war file
sudo ant inst   #Install the war file
```
Compile image generator:
```
cd /opt/TPC-W/java-tpcw/tpcw/ImgGen/ImgFiles
make
cd /opt/TPC-W/java-tpcw/
```


#### Populate the website
Now we need to populate. So on one webserver only run:
```
sudo ant gendb      # This takes a while to complete
sudo ant genimg     # it takes around 2 hours
```
Now you need to copy the generated images to the other websevers. I used the scp command on the server that owns the images:
```
scp -r /var/lib/tomcat7/webapps/tpcw/Images  <webserver2IP>:~/Images
```
and after, on the other webservers move the Images folder into the tpcw tomcat folder:
```
sudo mv Images /var/lib/tomcat7/webapps/tpcw/
sudo chown -R tomcat7:tomcat7 /var/lib/tomcat7/webapps/tpcw/Images     #Change the file owner
```

#### Test
Restart Tomcat
```
sudo service tomcat7 restart
``` 
and visit http://localhost:8080/tpcw/TPCW_home_interaction 
if it is not your local machine, change localhost with the IP address of one of your webserver (example for http://13.69.193.129:8080/tpcw/TPCW_home_interaction)  

If you want you can create a direct link to the webpage (http://localhost:8080/tpcw/) creating the index.html file inside /var/lib/tomcat6/webapps/tpcw/ in all your webservers. So open the file
```
sudo nano /var/lib/tomcat7/webapps/tpcw/index.html 
```
and write this :
```
<html>
<meta http-equiv="refresh" content="0; url=/tpcw/TPCW_home_interaction" />
</html>
```


## Workload
If no error, TPC-W has been installed successfully.
```
cd /opt/TPC-W/java-tpcw/dist/
sudo java rbe.RBE -EB rbe.EBTPCW1Factory 30 -OUT run1.m -RU 100 -MI 1000 -RD 100 -WWW http://localhost:8080/tpcw/ -CUST 10000 -ITEM 1000
```
To get more detail specification of parameters, please read tpc-w/dist/doc/readme-rbe.txt.
A matlab script will be generated in tpc-w/dist/. You can analysis it with auxiliary scripts in tpc-w/tpcw/matlab/.


