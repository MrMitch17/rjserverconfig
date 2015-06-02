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


#Questions


> How will you get your simple webpage’s code onto the machine?

I'm hosting the php code in a git repository which is then being pulled down as a task within Ansible.  If I was pushing one-off config files, I would include them in the ansible project to be pushed independantly from the git repo.  This can be cleaner by pulling the code into a seperate directory then symlinking.

```
- name: Pull RJ Code from git
    git: repo=https://github.com/MrMitch17/rjtestcode.git
         dest=/var/www/html
         accept_hostkey=True
```

> Assume the system you’re provisioning is going to be used in a critical production system.  How can you make sure it’s secure and highly available?

Security with keys and passwords would be encrypted in ansible.  Use jinja syntax to include the encrypted variables that need to be passed into the playbooks and config files.  Anything in PHP that needs to be secure like DB passwords should be writen out into an INI file using the jinja templating / encrypted ansible files.  There would also be playbooks for rotating keys for the defined hosts when needed.  To make it highly available, I would need to know a little about the underlying architecture, like what failures/changes are programmed for.  But assuming a loadbalancer, I would remove the host that is being updated from the LB in the first step and not move on until it is verified that the updated node is back into the LB.

> What would be different if you had to provision dozens of machines rather than just one?

The main difference would be adding the other hosts to the "all" group.  Then adding steps to update and validate one host at a time.  I would also break out an actual provisioning role and roles that are needed to push a minor change.  Also, knowing what machiens can be updated in what order depending on their roles would be programmed into the Playbook.


> What did the configuration management tool make easy?  What did it make difficult?

Everything was pretty stright forward with ansible.  It does not require a client / server setup so getting it up and running was rather quick.  It integated with vagrant out of the gate, so with "vagrant up", it also got everything it needed to connect to the vagrant machine and begin the provisioning process.

> Can you write any automated tests to ensure the configuration is correct?

Ansible has a test for verifying that a site returns a 200 and it can also check to make sure the site contains a specific string.  I'm using this to make sure the site is returning the expected MD5 for 'hello world' from the SQL server.  This can also be achieved outside of an Ansible core function with a more comprehensive test script, or a curl. 

```
#Test for MD5
  - name: Grab Web content to variable
    action: uri url=http://localhost return_content=yes
    register: webpage
  #Skipps if it returns the correct MD5
  - name: Test the MD5
    fail: msg='Did not return the MD5'
    when: "'5eb63bbbe01eeed093cb22bb8f5acdc3' not in webpage.content"
```


