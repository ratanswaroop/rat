#AIC Full CD Deployment Guide
==============================
FullCD project integrates OpsSimple deployment workflow with CI/CD build processes using Jenkins. 

The aim of the project is to build ability to deploy AIC sites end-to-end with Jenkins based on build pipelines. 

A pipeline is collection of jobs each doing a specific action of AIC deployment ex: bootstrap-KVMs, Setup-FUEL, install 
Non-OpenStack-RO. Each job is defined to be atomic deployment task from overall AIC deployment workflow (DG), and all the 
manual interventions around performing the task have been automated. 

These pipelines, Jobs and sub-jobs, give a clean approach for automation testing to retest and redeploy a specific action 
without having to restart from scratch. Having clear idea on how jenkins work will be helpfull for FULLCD deployment.

FullCD Workflow: 
~~~
1. prepare site specific Bootstrap_seed.yaml,opssimple_site.yaml,certs,FQDN's,build binaries etc(already finished in PRE-DG)
2. Setup a Jenkins VM on Seed node
3. Bootstrap seed kvm
4. Bootstrap LCP nodes , launch vms , install openstack , Non-OpenStack components and trigger validation tests
5. Reset environment if needed.
~~~
## Setup jenkins Slave	
~~~

1.	Clone https://gerrit.mtn5.cci.att.com/#/admin/projects/aic-opssimple-fullcd repository in local VM/laptop.
2.	Under jenkinsvm dir, update “jenkinsvm_meta_data.yaml”, network details with available PXE IP for “eth0” , Management IP (vlan 2001) for “eth1”, in case of Large Site.
3. Update hostname with <ENV>(eg.mtn11) in “jenkinsvm_user_data.yaml” and Copy whole jenkinsvm dir on to Seed Node.
~~~
##Login to Seed KVM and perform following steps.

###Download Jenkins QCOW2
    $ sudo ./fullcd.py jenkinsvm download http://mirrors-aic.it.att.com/files/CICD/jenkins/jenkinsvm.qcow2
###Bootstrap Jenkins VM
    $ sudo ./fullcd.py jenkinsvm bootstrap
###To Delete Jenkins VM (use when necessary)
   $ sudo ./fullcd.py jenkinsvm delete jenkinsvm
Contact CI/CD Team with IP address to register Jenkins Slave in Jenkins Master.
Note:
	Management IP for Jenkins VM should be accessible via SSH.
	
Trigger the Setup process
### 1.automatic deployment from jenkins
To start the site build, after setting up of Jenkins slave and master setup. 
-	Login to the Jenkins dashboard:  https://jenkins.mtn5.cci.att.com/view/FULL-CD/
-	Select “Full-CD” in the tabs and click on “FullCD-Depolylab-Pipeline” (https://jenkins.mtn5.cci.att.com/view/FULL-CD/job/FULLCD-Deploylab-Pipeline/) 
-	In the left-nav, click on “Build with Parameters”, fill in details as shown below and click on “Build” 
~~~
	Gerrit-Branch : AIC-Branch 3.0.2/3.5, e.g release/3.0.1, master      
	ENV-TYPE: lcp/devtest 
	ENV-NAME: MTN11
	CLONE_REPOS: TRUE 
	Jenkins_slave_name: MTN11
~~~
-	Check the active logs and build progress from Jenkins dashboard 
-	Various Jenkins jobs can be submitted, to deploy end-to-end submit “FullCD-DeployLab-Pipeline” if requirement is to just do a piece of overall deployment,
 and whole lab is already built with FullCD then you can submit any of the following with the proper build-paramters. 
 
 | **Deployment Task** |  | 
| --- | --- | 
| FULLCD-Update-Apollo | Fetching Config DataUpdate the configuration files bootstrap_fuelastute, bootstrap_fuelplugins, site.yaml | 
| FULLCD - Bootstrap-KVMs | Install the Seed KVM's and MAAS, OPSC,FUEL VMs | 
|FULLCD - Configure-Opssimple | Update Configuration and AIC packages for install |
|FULLCD - Deploy-Core-LCP | Deploy Core LCP (27-VMS) including Contrail, Trove, Designate |
| FULLCD - Deploy Additional-Plugins |	Deploy and Update LCM, Trove, Designate, Mistral|
| FULLCD - Deploy-Compute-KVMs | Computes with different profiles
| FULLCD - Deploy-Non-Openstack | RO, DCAE, ASTRA, UWA, Nagios (excluding ATT-Tools/MUAM)
| FULLCD - Deployment Validation | Tempest testing
| Dashboard |  Basic Jenkins GUI and log collection in place |
| LCM Integration |
### 2.manual deployment steps
we can even perform manual deployment task wise.

### 2.1 Update Apollo
In this Job we clone both aic-opssimple-sites and aic-opssimple-fullcd repos on jenkins vm
and run a playbook to update the apollo code , move all yaml files to seed node and rename bootstrap_seed.yaml as site.yaml

git clone -b $GERRIT_BRANCH ssh://jenkins@gerrit.mtn5.cci.att.com:29418/aic-opssimple-sites
git clone -b $GERRIT_BRANCH ssh://jenkins@gerrit.mtn5.cci.att.com:29418/aic-opssimple-fullcd

run the below playbook from jenkins VM
~~~
ansible-playbook -i inventory playbooks/seedkvm_updateapollo.yml --extra-vars="site_folder=$ENV_NAME" 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~

### 2.2 Seed KVM Create vms

In this Job we run below playbook from jenkins VM to create fuel , opssimple and maas VMs on seed node and configure fuel VM .
~~~
ansible-playbook -i inventory/ playbooks/seedkvm_createvms.yml 
~~~
###2.3 Bootstrap KVMs
In this Job we run a playbook from jenkins vm to bootrstrap all the operational KVMs
~~~
ansible-playbook -i inventory/ playbooks/seedkvm_startkvms.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
### 2.4 Configure Opssimple
In this Job we run below playboook from jenkins VM to configure opssimple VM  , import opssimple_sites.yaml file to the GUI and generate scripts.

~~~
ansible-playbook -i inventory/ playbooks/opscvm_updateopssimple.yml --extra-vars="site_folder=$ENV_NAME" 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
###2.5 Deploy core LCP
In this Job we run below playbook from jenkins vm to update KVMs , launch VMs for openstack env, assign all the roles, deploy openstack. 
~~~
ansible-playbook -i inventory/ playbooks/opscvm_deploycorelcp.yml 2>&1 | tee $WORKSPACE/$FILENAME  ; ( exit ${PIPESTATUS[0]} )
~~~
###2.6 Deploy additional Plugins
In this Job we run ansible playbook from jenkins vm to deploy additional fuel plugins
1.Launch LCM and TMDG VMs 
2.Assign roles and deploy them in the opestack env
~~~
ansible-playbook -i inventory/ playbooks/opscvm_deployaddplugins.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
### 2.7 Deploy compute
In this Job we run a playbook from jenkins vm to deploy compute
~~~
ansible-playbook -i inventory/ playbooks/opscvm_deploycompute.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~

###2.8 deploy swift bare metal
In this job we run a playbook from jenkins vm to deploy swift 
~~~
ansible-playbook -i inventory/ playbooks/opscvm_deployswift.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )

~~~
###2.9 Configure non-openstack
In this job we Configure Non-openstack components
~~~
ansible-playbook -i inventory/ playbooks/opscvm_configurenonopenstack.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
###2.10 LCM integration with non-openstack
In thsi job we integrate LCM with non-openstack
~~~
ansible-playbook -i inventory/ playbooks/setup_foreman_plugin.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
### 2.11 deployment validation
in this Job we validate the deployment
~~~
ansible-playbook -i inventory/ playbooks/fuelvm_executetempest.yml 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
~~~
###Note:
If Pipeline job fails, DE will need to re-run the FAILED Job after fixing the configuration if necessary and then continue executing individual downstream jobs only.

To Wipeout the Whole Lab:
~~~
1. Login to the Jenkins dashboard:  https://jenkins.mtn5.cci.att.com/view/FULL-CD/
2. Select “Full-CD” in the tabs and click on https://jenkins.mtn5.cci.att.com/view/FULL-CD/job/FULLCD-Wipeout-lab/ 
3. In the left-nav, click on “Build with Parameters”, fill in details as shown below and click on “Build” 

	FULLCD_DEPLOYED : Select TRUE if Lab deployed using FULLCD, else select FALSE
	Gerrit-Branch : AIC-Branch 3.0.2/3.5, e.g release/3.0.1, master      
	ENV-TYPE: lcp/devtest 
	ENV-NAME: MTN11
	Jenkins_slave_name: MTN11
~~~



