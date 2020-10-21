# Blog on Jenkins Pipeline to host website on Nginx server by provisioning infrastructure using Terraform on GCP instance

Hi all, In this blog we are going to see how we can provision infrastructure on GCP using terraform and how we automate this process by adding it to jenkins pipeline and deploying a html page on Nginx server installed on VM instance provisioned on the google cloud platform.

So lets start step by step 

## HTML file Upload to Git repo
First, Upload any simple static HTML page file you want to host on the server, it need not to be over complex because our main aim is to try to build **DevOps Pipeline**. 
I uploaded a simple file on my git repo. If you want you can refer to the same repo too for your own use: - [Git Repo Link](https://github.com/SHUBHAMBANSAL001/TRIAL_REPO)

 

## Jenkins Job Creation
Now Open your jenkins on your local machine in the directory where your **.war** file of jenkins is placed, using the command: 
```
java -jar jenkins.war
```
And your base home page will look like below.:

![Jenkins Dashboard](https://github.com/SHUBHAMBANSAL001/MIDSEM_SPCM/blob/main/IMAGES/JENKINS%20DASHBOARD.png "Jenkins Dashboard")

Now lets, create a New Job here for our automation work and name this project as **Automatepipeline** job.

![Automatepipeline Job](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/IMAGES/JENKINS%20JOB%20CREATION%20ZOOM.png "Automatepipeline Job")

Now as we have named this job, We have to configure it to access the code from git repo. \
(For this thing to perform you will need to install Git plugin inside Jenkins) \
After installing the plugin we connect the git repo to this job by configuring it under the **Source Code Management** section as shown below:

![Git Repo Connected](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/IMAGES/GIT%20REPO%20CONNECTED.png "Automatepipeline Job")

**Note**: Remember to put **.git** at the end of your repo URL because then its meaning will change from a URL to a git Repo

## Terraform Script Creation for Preparation of Infrastructure on Google Cloud Platform
Now We should Pause Jenkins Job Configuration here you can just leave it or Save it and resume later.

What we have done till now is we have linked the Git repo from which we will refer to the HTML code. Now our next goal is to provision infrastructure on Google Cloud platform(GCP). For this we will use **Terraform** tool in which we write scripts describing what resources we want to provision on cloud.

If you dont know about terraform and its working. You can refer the below link for tutorial: [Terraform Tutorial](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/Terraform_Tutorial.md)

Now go to workspace of your jenkins job you created just now by going on jenkins home directory. Create a new script file called **main.tf** as all of terraform script file end with **"tf"** format. Write the below code in the script for Instance creation on GCP.

```HCL
// Plugin or provider for accessing the resources on Google Cloud platform
provider "google" {
 credentials = file("credentials.json")
 project     = "${var.gcp_project}" 
 region      = "${var.region}"
}

// Code for Resource Instance
resource "google_compute_instance" "newinstance" {
  name         = "${var.name}-computeinstance"
  machine_type = "f1-micro" //With lowest memory as we are just practising
  zone         = "${var.zone}"

boot_disk {
    initialize_params {
      image = "debian-cloud/debian-9" //OS image it will boot 
    }
  }
  
  network_interface {
    network = "default"

    access_config {
      // Ephemeral IP
    }
  }
  
  metadata_startup_script = file("startup-script-custom")
}
```

As you can see in this code, first block which is **provider** block is used for connecting with Google Cloud project in which we are going to create our instance. Second block is of **Resource** which is basically for creation of our instance and arguments inside it for specific configuration of instance.  You can also see some variable which are not defined here because we will define them in another file called **variable.tf** file. The reason we define them in seperate file is because we dont have to define them again and again for every script we create inside the terraform directory so it becomes a bit easier for use to refer to them. 

So now we will create two files in the same directory as of the **main** script.
1) variables.tf (for declaring the variables which are going to be used)
2) terraform.tfvars (for defining the values of the variables which are declared)

**Note**: You might have notice one more file getting accessed called start up script. Leave that for now you will understand its purpose later in this activity.

Create a file called **variables.tf** and put the below code in it.
```HCL
variable "region" {}
variable "gcp_project" {}
variable "credentials" {}
variable "name" {}
variable "subnet_cidr" {}
variable "zone" {}
```

Now, Create another file called **terraform.tfvars** and put the below code in it:
```HCL 
region          = "us-central1"
gcp_project     = "vpc-project-terraform"
credentials     = "credentials.json"
name            = "shubham"
subnet_cidr     = "10.10.0.0/24"
zone		= "us-central1-a"
```
In this file you have to work a little more, here you have to provide values of your project id and then the credential file which you will download from your project. In the next phase we will download that key and refer it here for linking the script to the project on cloud platform.

So now our scripts are created but they are not done completely as you may have noticed that in the last steps we have to attach a key file called credentials.json which will be used to connect the script and cloud project. So lets generate that in next step

## Generate Credentials through Google Cloud Platform project
Go to google platform console and signup. You will get a free tier initially

If you dont know much about google cloud platform and wish to see an easy tutorial then you can access [this link](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/GCP_TUTORIAL.md)
Now create a new project in it and name it as you wish. and it will look like below:

![Google CLoud Console](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/IMAGES/GCP%20CONSOLE.png "Google CLoud Console")

Now you may see the project id in the screenshot above which you have to put in the terraform.tfvars file because what values were defined in that script were according to my project. \
Now lets generate the key for this project, Go to navigation menu on left and go to **IAM and Admin** area and click on **Service Accounts** inside it. Here we will create a new service account. and name it as you wish maybe like Sa-terraform.  After naming you will have to assign some **Role** to this account. For now you give it two roles 
1) Project Editor
2) Compute Admin

This will give the account power to access and control the compute resources in the project. Now it will ask you for generation of **key** of this account and save this key in the **json** format. Now this key will be downloaded to your local machine and rename it with name of **credentials.json**. Now move this file inside your jenkins job workspace in the same directory where your terraform script are residing.

Now just to get a brief about where we have reached till now, this whole step will provision resources on Google cloud platform.

## Startup Script for Nginx Server and its automated configuration on Compute Instance
As we have provisioned resources on Google Cloud platform using the scripts above, our work doesnt end here as now we still have to host the website on this instance. For that we need to do 2 things
1) Install and Run Nginx Server on the instance
2) Configure Nginx to host our html page. 

Both of these manual work can be automated if properly done , Now if you have done the tutorial of Terraform you might have noticed one feature there called **Startup Script** which are used just for this purpose where whenever we provision our environment we want some tools preinstalled on them. So we will take help of these scripts to install nginx server, configure it and run it.

Now if you look in the main script file there was a instruction regarding the start up script being included in it. This is where we are going to make changes and do all the necessary part. \
Open the file with the exact same we used in the script or you can customize as per your own use. \
If you wish to follow my naming convention then Create file named **startup-script-custom** and fill it with below code
```bash
#! /bin/bash

sudo apt-get -y install nginx
sudo service nginx start 
sudo service nginx status 

# Nginx has been installed till here now we will configure it to host website 

sudo su
apt-get -y install git
git clone "https://github.com/SHUBHAMBANSAL001/TRIAL_REPO.git"
cd TRIAL_REPO/


mkdir /var/www/shubham
cp page.html /var/www/shubham

cp shubham /etc/nginx/sites-available/
rm /etc/nginx/sites-available/default

ln -s /etc/nginx/sites-available/shubham /etc/nginx/sites-enabled/shubham
rm /etc/nginx/sites-enabled/default

service nginx restart
```

If you look into the script then its performing both of the steps we needed to do, i.e. we needed to first install the nginx server and then configure it to host our website. \
In case you don't know how to configure a server and you dont know what is working in the above script then you can refer this [link](https://medium.com/@jgefroh/a-guide-to-using-nginx-for-static-websites-d96a9d034940) 

So in this way nginx will be installed, our html code will be accessed by downloading code through open git repo. Then server will host it. 

## Configure jenkins job to run terraform scripts
Now that you have so many files in your jenkins job workspace, we should recall what we have in our directory.

--> Image or list of items inside the workspace on your local machine

Now you just have to configure your jenkins job using its **execute build** stage in which you give windows bash shell command in it.
Go to again your job and inside build department check option **execute windows bash command** option. \
Write the below code there. 
```
type page.html
terraform init 
terraform plan
terraform apply -auto-approve 
```

First command is just to show the contents of web page in your job console output so that you can confirm that what will be delivered on server in the end. And rest commands if you know how to use terraform are for provisioning the infrastructure.

![Jenkins Build Step](https://github.com/krupeshxebia/Xebia-Interns-2020/blob/shubham-bansal/IMAGES/JENKINS%20BUILD%20STEP.png "Jenkins Build Step")

Now just hit the save and apply button and come to dashboard of jenkins. Go to your job and press **Build now**.





## Workflow of the entire process
1) Create a jenkins Job
2) Attach a Sample HTML code to this Job by configuring it
3) Now go to workspace of your Job and Put Your terraform scripts (for infrastructure provisioning) there.
4) From the workspace using Command line Initialize the terraform script
5) This will create an environment on GCP and your terraform script will launch start script inside the instance
6) Using startup script in terraform, Install and launch an NGINX server on instance
7) Also using, Startup Script and Github Integration use to configure nginx server to host Website.
