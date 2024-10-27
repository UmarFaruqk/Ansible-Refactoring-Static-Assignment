## Ansible Refactoring & Static Assignments (Imports and Roles)
In this project, we will continue working with *ansible-config-mgt* repository and make some improvements of our code. Now we need to refactor our Ansible code, create assignments, and learn how to use the imports functionality. Imports allows us  to effectively re-use previously created playbooks in a new playbook - it allows us to organize our tasks and reuse them when needed.

Refactoring is a general term in computer programming. It means making changes to the source code without changing expected behaviour of the software. The main idea of refactoring is to enhance code readability, increase maintainability and extensibility, reduce complexity, add proper comments without affecting the logic.
Here we will move things around a little bit in the code, but the overal state of the infrastructure remains the same

# Step 1: Jenkins job enhancement
Before we begin, let us make some changes to our Jenkins job - now every new change in the codes creates a separate directory which is not very convenient when we want to run some commands from one place. Besides, it consumes space on Jenkins server with each subsequent change. Let us enhance it by introducing a new Jenkins project/job - we will require *Copy Artifact* plugin.
1. Go to your *Jenkins-Ansible* server and create a new directory called *ansible-config-artifacts* - we will store there all artifacts after each build. using *sudo mkdir /home/ubuntu/ansible-config-artifacts*
2. Change permissions to this directory, so Jenkins could save files there - *chmod -R 0777 /home/ubuntu/ansible-config-artifact* and also change the permission of the root directory using *chmod 775 /home/ubuntu* ![reference image](/Pictures/pic1.PNG) ![reference image](/Pictures/pic19.PNG) 
3. Go to Jenkins web console -> Manage Jenkins -> Manage Plugins -> on *Available* tab search for Copy Artifact and install this plugin without restarting Jenkins ![reference image](/Pictures/pic2.PNG)
4. Create a new Freestyle project (you have done it in Project 9) and name it *save_artifacts*. ![reference image](/Pictures/pic3.PNG) 
5. This project will be triggered by completion of your existing ansible project. Configure it accordingly ![reference image](/Pictures/pic4.PNG) ![reference image](/Pictures/pic5.PNG) ![reference image](/Pictures/pic6.PNG)

**NOTE**:You can configure number of builds to keep in order to save space on the server, for example, you might want to keep only last 2 or 5 build results. You can also make this change to your *ansible* job.
6. The main idea of *save_artifacts* project is to save artifacts into */home/ubuntu/ansible-config-artifacts* directory. To achieve this, create a Build step and choose *Copy artifacts* from other project, specify ansible as a source project and */home/ubuntu/ansible-config-artifacts* as a target directory. ![reference image](/Pictures/pic6.PNG)
7. Test your set up by making some change in README.MD file inside your *ansible-config-mgt* repository (right inside master/main branch) and you shall see your files inside */home/ubuntu/ansible-config-artifacts* directory if both jenkins job have been completed ![reference image](/Pictures/pic7.PNG) ![reference image](/Pictures/pic8.PNG)

# Step 2: Refactor Ansible code by importing other playbooks into *site.yml*
Before starting to refactor the codes, ensure that you have pulled down the latest code from master (main) branch, and create a new branch, name it *refactor*. using *git checkout -b refactor*
Most Ansible users learn the one-file approach first. However, breaking tasks up into different files is an excellent way to organize complex sets of tasks and reuse them.
1. Within *playbooks* folder, create a new file and name it *site.yml* - This file will now be considered as an entry point into the entire infrastructure configuration. Other *playbooks* will be included here as a reference. In other words, *site.yml* will become a parent to all other playbooks that will be developed. Including *common.yml* we created previously
2. Create a new folder in root of the repository and name it *static-assignments*. The **static-assignments** folder is where all other children playbooks will be stored. This is merely for easy organization of your work. It is not an Ansible specific concept, therefore you can choose how you want to organize your work.
3. Move *common.yml* file into the newly created *static-assignments* folder.
4. Inside *site.yml* file, import *common.yml* playbook. using the following code *---
- hosts: all
- import_playbook: ../static-assignments/common.yml* ![reference image](/Pictures/pic10.PNG) ![reference image](/Pictures/pic9.PNG)
5. Run *ansible-playbook* command against the *dev* environment
Since we need to apply some tasks to our dev servers and wireshark is already installed we wi;; create another playbook under *static-assignments* and name it *common-del.yml* in this playbook, configure deletion of *wireshark* utility. ![reference image](/Pictures/pic21.PNG)
6. update *site.yml* with - *import_playbook: ../static-assignments/common-del.yml* instead of *common.yml* and run it against dev servers using *cd /home/ubuntu/ansible-config-artifacts/* and *ansible-playbook -i inventory/dev.yml playbooks/site.yml* 
7. You should see this ![reference image](/Pictures/pic12.PNG) this ![reference image](/Pictures/pic13.PNG) then make sure wireshark is deleted on all servers by running which wireshark ![reference image](/Pictures/pic14.PNG)
Now we have learnt how to use *import_playbooks* module and we have a ready solution to install/delete packages on multiple servers with just one command.

# Step 3:  Configure UAT Webservers with a role 'Webserver'
We have our nice and clean *dev* environment, so let us put it aside and configure 2 new Web Servers as *uat*. We could write tasks to configure Web Servers in the same playbook, but it would be too messy, instead, we will use a dedicated role to make our configuration reusable.
1. Launch 2 fresh EC2 instances using RHEL 8 image, we will use them as our *uat* servers, so give them names accordingly - *Web1-UAT* and *Web2-UAT*.
**Tip**:  Do not forget to stop EC2 instances that you are not using at the moment to avoid paying extra. For now, you only need 2 new RHEL 8 servers as Web Servers and 1 existing *Jenkins-Ansible* server up and running.
2. To create a role, you must create a directory called *roles/*, relative to the playbook file or in */etc/ansible/* directory.
There are two ways how you can create this folder structure:
- Use an Ansible utility called *ansible-galaxy* inside *ansible-config-mgt/roles* directory (you need to create roles directory upfront) using *mkdir roles* *cd roles* then *ansible-galaxy init webserver*
- Create the directory/files structure manually and the entire sfolder should look like this ![reference image](/Pictures/pic11.PNG)
3. Update your inventory *ansible-config-artifacts/inventory/uat.yml file with IP addresses of your 2 UAT Web servers
**NOTE**:  Ensure you are using ssh-agent to ssh into the Jenkins-Ansible instance just as you have done in project 11 ![reference image](/Pictures/pic20.PNG)
3. In */etc/ansible/ansible.cfg* update the *roles path* string and provide a full path to your roles directory ![reference image](/Pictures/pic22.PNG)
4. It is time to start adding some logic to the webserver role. Go into tasks directory, and within the *main.yml* file, start writing configuration tasks to do the following
- Install and configure Apache (httpd service)
- Clone Tooling website from GitHub *https://github.com/<your-name>/tooling.git*.
- Ensure the tooling website code is deployed to */var/www/html* on each of 2 UAT Web servers.
- Make sure httpd service is started

# Step 4: Reference 'Webserver' role
1. Within the *static-assignments* folder, create a new assignment for **uat-webservers** *uat-webservers.yml*. This is where you will reference the role. ![reference image](/Pictures/pic23.PNG)
2. Remember that the entry point to our ansible configuration is the *site.yml* file. Therefore, you need to refer your *uat-webservers.yml* role inside *site.yml*. And we should have this ![reference image](/Pictures/pic24.PNG)

# Step 5:  Commit & Test
1. Commit your changes, create a Pull Request and merge them to master branch, make sure webhook triggered two consequent Jenkins jobs, they ran successfully and copied all the files to your Jenkins-Ansible server into */home/ubuntu/ansible-config-artifacts/* directory.
2.  run the playbook against your *uat* inventory and see what happens using *cd /home/ubuntu/ansible-config-artifacts* and *ansible-playbook -i /inventory/uat.yml playbooks/site.yaml*

**NOTE**: Before running your playbook, ensure you have tunneled into your Jenkins-Ansible server via ssh-agent
3. You should see this ![reference image](/Pictures/pic16.PNG) ![reference image](/Pictures/pic15.PNG)
4. reach your UAT Webservers from your browser using *http://<Web1-UAT-Server-Public-IP-or-Public-DNS-Name>/index.php* and you should see this ![reference image](/Pictures/pic17.PNG)
**NOTE**: Don't forget to enable the HTTP port 80 in your inbound rules.
5. Now our architecture looks like this ![reference image](/Pictures/pic18.png)

**CONGRATULATIONS!**: We have learnt how TO  deploy and configure UAT Web Servers using Ansible imports and roles!
