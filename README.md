# Kerberos Setup on Cloudera Quickstart

### 1. Start the docker container 

Some ports are required to be open so that you Docker container can communicate with the "outside world".  
Specifically we need to make sure that these ports are released on your localhost machine or atleast map them to other ports:  
- cloudera welcome tutorial - 80  
- oozie - 8888  
- cloudera manager - 7180  
- hdfs REST api- 8020  
- kerberos - 88 (both tcp and udp)  

Also try to use atleast 8GB of RAM memory.

	docker run --hostname=quickstart.cloudera --privileged=true -t -i -m 14G -p 80:80 -p 8888:8888 -p 7180:7180 -p 8020:8020  -p 88:88 -p 88:88/udp [IMAGE ID] /usr/bin/docker-quickstart


---

### 2. Re-start Cloudera Manager and services (if it did not start properly)

	sudo /home/cloudera/cloudera-manager --express --force;  sudo service cloudera-scm-server restart

- this will take 3-4 minutes to finish

Now open the Cloudera manager and restart all services (hdfs, yarn, hue...) by using a single button.  


---

### 3. Run the default Kerberos configuration (If not installed) 

Run this script: 

	/home/cloudera/kerberos

-  here you don’t have to do anything except remember the output of the script where they provide crucial information for later installation

- if it fails during installation (with this error: Rpmdb checksum is invalid: dCDPT(pkg checksums): libss.x86_64 0:1.41.12-23.el6 - u) , just re-run it and it should work

**You will get this output**:

Success! Kerberos is now running. You can enable Kerberos in a Cloudera Manager
cluster from the drop-down menu for that cluster on the CM home page. It will
ask you to confirm that this script performed the following steps:

    * set up a working KDC.
    * checked that the KDC allows renewable tickets.
    * installed the client libraries.
    * created a proper account for Cloudera Manager.

Then, it will prompt you for the following details (accept defaults if not
specified here):

    KDC Type:                MIT KDC
    KDC Server Host:         quickstart.cloudera
    Kerberos Security Realm: CLOUDERA

Later, it will prompt you for KDC account manager credentials:

    Username: cloudera-scm/admin (@ CLOUDERA)
    Password: cloudera


---

### 4. Enable Kerberos in Cloudera Manager (If not installed) 

Open in your browser: **localhost:7180**

To login into Cloudera Manager use:   

	username = admin
	password = admin
	

Now go to **Administration** tab and choose **Security**.  
Then click **Enable Kerberos**.

1. Check all 4 of the boxes as they were all created in the previous step.

2. In the next step:  
	1) Enter the	KDC Server Host from the script (quickstart.cloudera)  
	2) Change the Kerberos Security Realm to the one provided in the script  (CLOUDERA) 

3. In the next step:  
check the ***Manage krb5.conf through Cloudera Manager*** Box and Continue. 


4. In the next step:  
enter the Username and Password from the script (cloudera-scm/admin and cloudera)


5. In the next step:  
wait for  **Import KDC Account Manager Credentials Command** to finish and Continue.

6. In the last step choose to Restart the cluster.  


	**If the last step of Restart hangs, try to refresh the page, if there is no response, you may need to restart the Cloudera Quickstart Manager manually using this command (it will take 2-3 minutes to finish)**  
			
		sudo service cloudera-scm-server restart
		
7. You can check that both Kerberos and Cloudera Manager are working by checking ports 88 and 7180 using this command:
		
		sudo netstat -tulpn | grep 88  
	
	and 

		sudo netstat -tulpn | grep 7180

8. Lastly you just have to restart all of the services in Cloudera Manager.  
Or atleast only restart Zookeeper, YARN and HDFS (in that order). 

9. Now if you try the command 

		hdfs dfs -ls /
		
You should be blocked by Kerberos raising a PrivilegedActionException.


---

### 5.Adding a Kerberos user

You have to be using the "root" user to do the following.  

Enter the Kerberos interactive shell type 

	kadmin.local
		
Now add a principal with name "test" by typing this command and then providing a password for this user  

	addprinc test@CLOUDERA
		
Please remember the pricipal's password as your clients will login using that.  
To check that the principal was created, run: 

	list_principals

You can test the login with that user by exiting the kadmin.local shell and typing: 

	kinit test@CLOUDERA 
	
Then if you type "klist" you will see that there is a ticker using which you are logged in.
You can also delete that ticked using "kdestry" command and then you can re-login.  

- 
You can also export the keytab of a principal using  

	xst -k test.keytab test@CLOUDERA
	
and then copy the keytab outside of the container into your PC, eg. 

	docker cp [container_id]:/test.keytab /Users/dorianbeganovic/Desktop/

Now you can finally login as test@CLOUDERA using Kerberos at your local (client) machine by using  

	kinit -k -t /Users/dorianbeganovic/Desktop/test.keytab test@CLOUDERA

Now we need to add a HDFS folder for that user.

---

### 6.Creating a Hadoop folder for a Kerberos user

After you added the user to hdfs, the user will have no write access unless you do the following: 


1. Establish the hdfs principal (If not already established)  
2. Create a folder with user ownership

##### Establishing hdfs principal  (If not already established)
Write 
	
	kadmin.local

Then add user with 

	addprinc hdfs@CLOUDERA

Then exit the kadmin.local shell with 

	exit

Lastly write 

	kinit hdfs@CLOUDERA 

to login to kerberos as hdfs principal which gives you the full write permissions. 


##### Creating a folder for user

You must first be logged in as **hdfs** principal (by running kinit command).  
Now let's say the user you added is test@CLOUDERA.  
The hadoop command you have to run is :  

	hadoop fs -mkdir /user/test
	hadoop fs -chown test /user/test  

With this, you can now access the HDFS !

--- 

### 7. Configuring Kerberos on a client machine

There are a few easy steps in this process.  

For windows configuration you can also look at this manual http://doc.mapr.com/display/MapR/Configuring+Kerberos+Authentication+for+Windows  
or pages 44-46 of this manual http://hortonworks.com/wp-content/uploads/2014/05/Product-Guide-HDP-2.1-v1.01.pdf  


1. Add "147.228.63.46   quickstart.cloudera" entry to your /etc/hosts file  

On linux you can use:
		
	echo "147.228.63.46   quickstart.cloudera" >> /etc/hosts
		
Or on windows use: 

	echo 147.228.63.46   quickstart.cloudera >> %WINDIR%\System32\Drivers\Etc\Hosts
	ipconfig /flushdns

2. Install client application: 

On linux you have to install the krb5-libs and krb5-workstation packages:
		
	 yum install krb5-workstation krb5-libs 
	 
For ubuntu it is a bit different, but at the bottom of this page there is a good manual: 

	https://help.ubuntu.com/lts/serverguide/kerberos.html
	 
On windows: 

	On 64-bit machines install this program http://web.mit.edu/kerberos/dist/kfw/4.0/kfw-4.0.1-amd64.msi

3. Obtain krb5.conf file from your administrator.

To set up the Kerberos configuration file in the default location, obtain the krb5.conf configuration file from your Kerberos administrator.

On linux you have to copy the krb5.conf file you received into the /etc/ folder (overwrite the default file).  
So run the command:  
		
	cp /.../krb5.conf /etc/

On windows you should:

	Rename the configuration file from krb5.conf to krb5.ini.
	Copy the krb5.ini file to the C:\ProgramData\MIT\Kerberos5 directory, and overwrite the empty sample file.

4. Login to Kerberos 

On linux just write: 

	kinit username@CLOUDERA

And provide the proper password.

On windows it is within a GUI so please refer to the manuals I specified above.  

---


### 8. Accessing HDFS of a Hadoop cluster with Kerberos authentication  (Development)

**1. Using the Hadoop shell**

If you are on the local machine, you can just initialize the Hadoop cluster using ***kinit*** and it will work. 

If you want to use the shell with a remote cluster, then you should update your  hdfs-site.xml and core-site.xml files by downloading the configuration from Cloudera Manager and placing it in your proper folder. 

**2. Using the Java API**

The method I use is using an already initialized Kerberos account.  
This means that you first login to Kerberos using the kinit command.
Then in your Java code use this snippet when first accessing HDFS:


    Configuration conf = new Configuration();
    conf.set("hadoop.security.authentication", "kerberos");
    conf.set("dfs.namenode.kerberos.principal.pattern", "hdfs/quickstart.cloudera@CLOUDERA");
	
	UserGroupInformation.setConfiguration(conf);
	# uses the background kinit 
	UserGroupInformation.getLoginUser().checkTGTAndReloginFromKeytab();
	
	fileSystem = FileSystem.get(URI.create(uriPrefix + homeDirectory), conf);

This will use the Kerberos authentication you started using the **kinit** command. 


You can also login using a Keytab within the Java code.  
This is done using:   
		
	UserGroupInformation.loginUserFromKeytab(String user,String keytabPath)
	

But I found that this method did not work and was harder to deploy.  


