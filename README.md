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

### 2. Re-start Cloudera Manager (if it did not start properly)

	sudo /home/cloudera/cloudera-manager --express --force;  sudo service cloudera-scm-	server restart

- this will take 3-4 minutes to finish

---

### 3. Run the default Kerberos configuration 

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

### 4. Enable Kerberos in Cloudera Manager

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

### 5.Add a Kerberos user

To enter the Kerberos interactive shell type 

	kadmin.local
		
Now add a principal by typing and providing a password   

	addprinc test@CLOUDERA
		
Export the keytab of a principal using  

	xst -k test.keytab test@CLOUDERA
	
Then copy the keytab outside of the container into your PC, eg. 

	docker cp [container_id]:/test.keytab /Users/dorianbeganovic/Desktop/

Now you can finally login as test@CLOUDERA using Kerberos at your local (client) machine by using  

	kinit -k -t /Users/dorianbeganovic/Desktop/test.keytab test@CLOUDERA


With this, you can now safely access the Hadoop cluster !

---


### 6. Accessing Hadoop cluster with Kerberos authentication 

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


