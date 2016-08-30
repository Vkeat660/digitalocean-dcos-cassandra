# Quickstart Tutorial: Cassandra Cluster on Digital Ocean using CoreOS #

This tutorial is a quick and dirty guide for getting a cassandra cluster up and running in less than an hour.

While beyond the scope of this tutorial, it's also pretty straightfoward to install Spark, Kafka, or whatever else you need on dc/os once the cluster is up and running.

## Tech Stack ##

- Terraform
- Digital Ocean
- DC/OS (Apache Mesos)
- Cassandra

## Requirements ##

- A digital ocean account with API token and 10+ droplet limit
- OSX, or a similar OS with unix command shell


## Initial Steps ##

If you haven't yet generated your digital ocean API token, go ahead and do that now. Instructions can be found [here](https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2)

We will also need an ssh key which can be generated with the following in terminal.

```
ssh-keygen -t rsa -P '' -f ~/.ssh/dcos-key
chmod 600 ~/.ssh/dcos-key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/dcos-key
```

In this tutorial I'm going to use the convention **[Digital Ocean API Key]** to signal when you need to replace with your own secret information, and we will be hardcoding values for simplicity. You will have to replace 2 values in the next command.

I will also be using 000.000.00.000 in place of actual IPs

```
curl -X POST -H "Content-Type: application/json" -H "Authorization: Bearer [Digital Ocean API Key]" -d '{"name":"dcos-key","public_key":"[Public Key]"}' "https://api.digitalocean.com/v2/account/keys"
```

where **[Public Key]** is the contents of **~/.ssh/dcos-key.pub**. For the lazy among us:

```
pbcopy < ~/.ssh/dcos-key.pub
```

In the json you get back from the server, identify the record with your key name and look for the ssh key fingerprint, which looks similar to "00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00". Record this value for use later.

You can see all your keys and verify the information by using:

```
curl -X GET -H 'Content-Type: application/json' -H 'Authorization: Bearer [Digital Ocean API Key]' "https://api.digitalocean.com/v2/account/keys"
```


## Setting up terraform ##

Clone the following repo, and open terraform.tfvars in your favorite text editor.

```
git clone https://github.com/Vkeat660/digitalocean-dcos-cassandra.git
cd digitalocean-dcos-cassandra
```

You'll need to replace 4 values from this file that were generated in the "Initial Steps" section.

```
digitalocean_token = "[Digital Ocean API Key]"
dcos_ssh_key_path = "~/.ssh/dcos-key"
dcos_ssh_public_key_path = "~/.ssh/dcos-key.pub"
ssh_key_fingerprint = "[SSH Key Fingerprint]"
```

Also, you may need to update

```
dcos_installer_url = "https://downloads.dcos.io/dcos/EarlyAccess/commit/14509fe1e7899f439527fb39867194c7a425c771/dcos_generate_config.sh"
```

which can be found on page: [dc/os instructions for installing on packet](https://dcos.io/docs/1.7/administration/installing/cloud/packet/)

Finally, save the file and run:

```
terraform apply
```

This step takes a few minutes to complete. It will give you the following output:

```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

Outputs:

Use this link to access DCOS = http://000.000.00.000/
agent-ip = 000.000.00.000,000.000.00.000,000.000.00.000,000.000.00.000
agent-public-ip = 000.000.00.000,000.000.00.000,000.000.00.000
bootstrap-ip = 000.000.00.000
master-ip = 000.000.00.000,000.000.00.000,000.000.00.000
```

If at anytime you need to destroy the cluster, run the command:

```
terraform destroy
```


## Troubleshooting Intermission 1 ##

If you don't get this output, it might be one of the following problems:

- Your digital ocean account doesn't have permission to create 10 droplets
- The value of dcos_installer_url has been changed since I created this tutorial
- Syntax error when updating terraform.tfvars
- You've ran terraform before and need to clear out some of the files it generated to match the original repo


## Installing Cassandra on DC/OS ##

In the output from terraform you should have been given a link, but likely it will need 5-10 minutes for it to work. Go grab your favorite caffinated beverage.

Once it starts working, login with your github account to create the admin account.

We will need to install dcos command line tools, and the easiest way is from the dashboard page. In the lower left corner there should be your email and a "drop up" to reveal the Install CLI Option with commands to install.

Importantly, it will also do some configuration to point to the server we just created.

Type dcos for a list of commands:

```
dcos
```

Login to the service using:

```
dcos auth login
```

And follow the prompts to open up a web browser and authenticate.

Finally run

```
dcos package install cassandra
```

and after a few minutes, to verify the install

```
dcos cassandra connection
```

If the installation completed it should look like:

```
{
  "address": [
    "00.000.00.000:9042",
    "00.000.00.000:9042",
    "00.000.00.000:9042"
  ],
  "dns": [
    "node-0.cassandra.mesos:9042",
    "node-1.cassandra.mesos:9042",
    "node-2.cassandra.mesos:9042"
  ]
}
```


## Troubleshooting Intermission 2 ##

If you see the following after runing dcos cassandra connection:

```
{
  "address": [],
  "dns": []
}
```

It means that dcos marathon wasn't able to complete the installation task. In my case this was because my nodes only had 4GB of ram initially, but when I increased to 8GB the problem resolved itself.

This may be a good time to familiarize yourself with marathon and how to access logs in order to diagnos problems, since similar issues can occur with installing spark and kafka.

Other issues can occur if you skipped authenticaion using **dcos auth login**, or have since moved to another bash shell window and forgot to reauthenticate.

## Testing that the installation works! ##

Ok, now for the fun part haha.

If you added the ssh key to your agent earlier you should be able to just type:

```
dcos node ssh --master-proxy --leader
docker run -ti cassandra:3.0.7 cqlsh [Cassandra Node IP]
```

where the [Cassandra Node IP] is any of the IPs that we got from running **dcos cassandra connection**

If you haven't used cassandra before, here's some commands you can run in cqlsh to populate the data:

```
CREATE KEYSPACE demo WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };

USE demo;CREATE TABLE map (key varchar, value varchar, PRIMARY KEY(key));

INSERT INTO demo.map(key, value) VALUES('Kimchi', 'Tasty');
INSERT INTO demo.map(key, value) VALUES('Vivian', 'Awesome');
INSERT INTO demo.map(key, value) VALUES('Devops', 'Fun!');

SELECT * FROM demo.map;
```


## Where to go from here ##

Now that you've got a cassandra cluster up and running, it's relatively easy to add injestion and analytics on top of it. Although not covered in this tutorial, you might want to try to following:

```
dcos package install spark
dcos package install kafka
```

Just watch out that you may need to spin up more machines to handle the load (it can be done easily with terraform)

Thanks! And let me know in issues if you have any problems.
