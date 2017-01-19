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
Clone both aic-opssimple-sites and aic-opssimple-fullcd repos on jenkins vm
git clone -b $GERRIT_BRANCH ssh://jenkins@gerrit.mtn5.cci.att.com:29418/aic-opssimple-sites
git clone -b $GERRIT_BRANCH ssh://jenkins@gerrit.mtn5.cci.att.com:29418/aic-opssimple-fullcd

run the below playbook from jenkins VM
ansible-playbook -i inventory playbooks/seedkvm_updateapollo.yml --extra-vars="site_folder=$ENV_NAME" 2>&1 | tee $WORKSPACE/$FILENAME ; ( exit ${PIPESTATUS[0]} )
#### task 1: Update apollo code
sudo apt-get update aic-opssimple-apollo && apt-get upgrade aic-opssimple-apollo
#### task 2: Update the configuration files from Jenkins
move all the following yamls files from jenkins vm to seed node
~~~
           - bootstrap_fuelastute.yaml
           - bootstrap_fuelplugins.yaml
           - bootstrap_maasvm.yaml
           - bootstrap_opscvm.yaml
           - bootstrap_seed.yaml
~~~
#### task 3: Rename the bootstrap_seed.yaml to site.yaml
on seed node run following command
mv /home/ubuntu/apollo/bootstrap_seed.yaml /home/ubuntu/apollo/site.yaml

### 2.2 Seed KVM Create vms


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



