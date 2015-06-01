# RJ-Ansible Demo
This project will provision a server using Ansible on a local Ubuntu Vagrant server and pull code from a git repo.


Needed Tools
- Vagrant
- Ansible

# What does it do?

1) Install the base system and needed packages.
    - apache2
    - git-core
    - python-mysqldb
    - mysql-server
    - php5
    - libapache2-mod-php5
    - php5-mysql

2) Set up / configure the server to serve the site.

```
---
  #Create a MySQL User and password.  For this it just allows all privs
  - name: Setup MySQL DB
    mysql_db: name=rj state=present
  - name: Setup MySQL User
    mysql_user: name={{ mysqluser }} password={{ mysqlpassword }} priv=*.*:ALL state=present

  #Ignoring Setup of Users and Perms or Symlinks Delete the /var/www/html 
  - name: Remove /var/www/html
    file: path=/var/www state=absent
  

  #Pull Code from Git into /var/www/html
  - name: Pull RJ Code from git
    git: repo=https://github.com/MrMitch17/rjtestcode.git
         dest=/var/www/html
         accept_hostkey=True
  #Reload Apache2
  - name: Reload Apache to serve new code
    service: name=apache2 state=reloaded
  
  #Test for MD5
  - name: Grab Web content to variable
    action: uri url=http://localhost return_content=yes
    register: webpage
  #Skipps if it returns the correct MD5
    name: Test the MD5
    fail: msg='Did not return the MD5'
    when: "'5eb63bbbe01eeed093cb22bb8f5acdc3' not in webpage.content"

```


#Start the system

To provision the vagrant server all that is needed is to run 
```sh 
vagrant up
```

Test by going to http://localhost:8080 . It works if "5eb63bbbe01eeed093cb22bb8f5acdc3" is displayed showing that a valid MySQL md5() query has been issued against the server


The following ports are mapped to the local machine:
- 22 => 2222
- 80 => 8080

##Improvements
- This is currently deleting the /var/www/html directory in preparation for the git pull. This causes the website to be unavailable during this operation. To make this highly available, when pushing production code, I would remove the webserver being updated out of the load-balancer while it is being updated.
- This is for a single system.  When Deploying a production version of this, I would break out the roles into more defined roles other than "base" and "everything else"  These roles were created to show the use of roles in ansible. That way we can pull new code and reload the site independently from the other commands. Also, better defined hosts.
- I am passing the MySQL db user / pass using variables.  Currently the user/pass is also hardcoded into the php page. To increase security, I would use encrypted ansible files  to write out a .ini file using variables and have php read the .ini

