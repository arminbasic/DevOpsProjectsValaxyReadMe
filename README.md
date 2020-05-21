# DevOpsProjectsValaxyReadMe

DevOps Projects

PROJECT 1:

- Launch 2 EC2 instances:

 	Inst_1 - 1 instance for Web server: Red Hat Enterprise Linux 8 (HVM), SSD Volume Type, General Purpose t2.micro (new Security Group) - install tomcat 
 	Inst_2 - 1 instance for Jenkins server: Red Hat Enterprise Linux 8 (HVM), SSD Volume Type, General Purpose t2.micro (new Security Group) - install jenkins  

- Install java on both instances : yum install java 1.8* -y
- Install wget on both instances : yum -y install wget

- Both Security Groups modified to look like this:
	Inbound rules:
SSH	TCP	22	0.0.0.0/0	-
Custom TCP	TCP	8090	0.0.0.0/0	-
Custom TCP	TCP	8090	::/0	-

	Outbound rules:
All traffic	All	All	0.0.0.0/0	-

Inst_1:

- Install tomcat on Inst_1: 
	cd /opt
	wget https://apache.mirror.ba/tomcat/tomcat-9/v9.0.35/bin/apache-tomcat-9.0.35.tar.gz
	tar -xvzf apache-tomcat-9.0.35.tar.gz
- Changing permissions in order to execute .startup.sh and shutdown.sh
	chmod +x /opt/apache-tomcat-8.5.35/bin/startup.sh shutdown.sh
- Creating soft links for starting (tomcatup) and shuting down (tomcatdown) tomcat:
	ln -s /opt/apache-tomcat-8.5.35/bin/startup.sh /usr/local/bin/tomcatup
  	ln -s /opt/apache-tomcat-8.5.35/bin/shutdown.sh /usr/local/bin/tomcatdown
- Change port number (conf/server.xml file): 
	cd apache-tomcat-9.0.34/conf
	vi server.xml
- Edit file : Change port number from 8080 to 8090
- Update tomcat users file (conf/tomcat-user.xml):
	cd apache-tomcat-9.0.34/conf
	vi tomcat-user.xml
- Add roles and users:
	<role rolename="manager-gui"/>
	<role rolename="manager-script"/>
	<role rolename="manager-jmx"/>
	<role rolename="manager-status"/>
	<user username="admin" password="admin" roles="manager-gui, manager-script, manager-jmx, manager-status"/>
	<user username="deployer" password="deployer" roles="manager-script"/>
	<user username="tomcat" password="s3cret" roles="manager-gui"/>


Inst_2:

- Edit bash_profile:
	vi ~/.bash_profile
 
	JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el8_1.x86_64
	PATH=$PATH:$JAVA_HOME:$HOME/bin
	export PATH

	source ~/.bash_profile 

- Install Jenkins on Inst_2:
	yum -y install wget
	wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
	rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
	yum -y install jenkins

-Start jenkins : systemctl start jenkins (for stop: systemctl stop jenkins)

- Access jenkins on : http://PUBLIC-IP:8080
- Login to jenkins:  Username: admin password: on Location:/var/lib/jenkins/secrets/initialAdminPassword (use cat command on this file), you can add new user after login

- Install plugins: Github, Maven invoker, deploy to container (install without restart)

- Setup credetnials for deployment:
	Credentials (on side) > global (under Domain) > Add Credentials
	Username : deployer
	Password : deployer
	id : Tomcat_user
	Description: Tomcat user to deploy on tomcat server

- Create Jenkins Job, Configure it like this:
	Source Code Management:
		Repository: https://github.com/ValaxyTech/hello-world.git
		Branches to build : */master
	Build:
		Maven Version : Maven		
 		Goals:	clean install package
 		POM: pom.xml

	Post-build Actions: 
		Deploy war/ear to container
			WAR/EAR files : **/*.war
		     Containers : Tomcat 8.x
			Credentials: Tomcat_user (created recently)
			Tomcat URL : http://PUBLIC_IP:PORT_NO

- Add Build Triggers for CI/CD : -configure jenkins job to:
	 Build Triggers
		Poll SCM
			Schedule H/2 * * * *

PROJECT 2:

- Launch new instance : 
	Inst_2 : 1 instance for Jenkins server: Red Hat Enterprise Linux 8 (HVM), SSD Volume Type, General Purpose t2.micro (new Security Group) - install ansible

- Install ansible on Inst_3 : 
	rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
	yum install ansible -y

- Add user and assign password:
	useradd ansadmin
	passwd ansadmin

- Modify sudoers file, add among users:
	ansadmin ALL=(ALL)       NOPASSWD: ALL

- Add hosts to hosts file (/etc/ansible/hosts)
	cd /etc/ansible/hosts
	vi hosts

	Add tagrget server IP address

- Modify ssh_config file (Password configuration) for keybased authentication:
	vi /etc/ssh/sshd_config
	PasswordAuthentication yes (no to yes)

- Generate key: ssh-keygen
- Copy key for client : ssh-copy-id <target-server-IP-address>
 

- Install plugin : publish Over SSH

- Enable connection between Jenkins server and Ansible server, Add Ansible server
 	Manage Jenkins > Configure System > Publish Over SSH > SSH Servers > Add
	
	SSH Servers:
	Hostname:<ServerIPAddress>
	username: ansadmin
	password: *****

- Create Ansible playbook on Inst_3:
---
- hosts: all 
  become: true
  tasks: 
    - name: copy jar/war onto tomcat servers
        copy:
          src: /op/playbooks/wabapp/target/webapp.war
          dest: /opt/apache-tomcat-8.5.32/webapps

- Configure job on Jenkins server and add following to current configurations:
	Add post-build steps

	    Send files or execute commands over SSH
		SSH Server : ansible_server
		Source fiels: webapp/target/*.war
		Remote directory: //opt//playbooks
		Add post-build steps

	    Send files or execute commands over SSH
		SSH Server : ansible_server
		Exec command ansible-playbook /opt/playbooks/copywarfile.yml

PROJECT 3: 


- Launch new EC2 instance:
	Inst_4 : 1 instance for Docker server: Red Hat Enterprise Linux 8 (HVM), SSD Volume Type, General Purpose t2.micro (new Security Group)

- Install Docker on Inst_4 and start service:
	yum install docker
	service docker start

- Add user for docker managment and create password for it and add user in group (if group does not exist run: groupadd GroupName)
	useradd dockeradmin
	passwd dockeradmin
	usermod -aG docker dockeradmin

- Write in DockerFile:
	cd opt/
	mkdir docker
	cd docker 
	vi Dockerfile

	From tomcat:8-jre8 
	# Maintainer
	MAINTAINER "valaxytech" 
	# copy war file on to container 
	COPY ./webapp.war /usr/local/tomcat/webapps

- On Jenkins server, add Docker server:
	Manage Jenkins > Configure system > Publish over SSH > Add server (docker)

- Create new Jenkins job and configure it to look like this: 
	Source Code Management
		Repository : https://github.com/ValaxyTech/hello-world.git
		Branches to build : */master

	Build Root POM: pom.xml
		Goals and options : clean install package

	send files or execute commands over SSH Name: docker_host
		Source files : webapp/target/*.war Remove prefix : webapp/target Remote directory : //opt//docker
		Exec command[s] :
		docker stop valaxy_demo;
		docker rm -f valaxy_demo;
		docker image rm -f valaxy_demo;
		cd /opt/docker;
		docker build -t valaxy_demo .

	send files or execute commands over SSH
		Name: docker_host
		Exec command : docker run -d --name valaxy_demo -p 8090:8080 valaxy_demo

- Check docker images and containers before and after execution of Jenkins job:
	cd opt/docker
	docker images
	docker ps
	

