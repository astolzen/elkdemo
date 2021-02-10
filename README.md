# Ansible Tower full Application Rollout Demo with Elasticsearch, Logstash and Twitter

*v0.91 - 2/10/21*

# What is this
A collection of Ansible Playbooks and Roles to roll out an ELK (Elasticsearch, Logstash, Kibana) Stack on two virtual Machines.

 The Logstash VM will connect to **Twitter** and collect all Tweets to a given Keyword. The Access to the Kibana Web-UI is proxied through Nginx, secured which a *htpasswd*-Authentication.

This Demo is intended to run as a workflow from within Ansible Tower (or AWX).


# Dependencies
## Custom Credential Types
This Demo requires Custom Credential Types for
 * Dynamic DNS Updates using "nsupdate"
 * The Twitter API-Key (get your Key here: https://developer.twitter.com)
 * SMTP-Credentials to send E-Mails

The DDNS- and Sendmail-Role are optional and can be left out or replaced with different Roles.
The custom credentials used in this Demo are defined like this:

### DDNS Credential Type
<details>
  <summary markdown="span">DDNS Credentials</summary>

#### Input Configuration

```
fields:
  - id: ddnskey
    type: string
    label: DDNS Key Name
  - id: ddnszone
    type: string
    label: DDNS Zune Name
  - id: ddnsserver
    type: string
    label: DDNS Server
  - id: ddnssecret
    type: string
    label: DDNS Secret
    secret: true
required:
  - ddnskey
  - ddnszone
  - ddnsserver
  - ddnssecret
```
#### Injector Configuration
```
extra_vars:
  ddns_key_name: '{{ ddnskey }}'
  ddns_key_secret: '{{ ddnssecret }}'
  ddns_key_server: '{{ ddnsserver }}'
  ddns_key_zone: '{{ ddnszone }}'
```
</details>

#### Twitter API
<details>
  <summary markdown="span"> Twitter API Credentials</summary>

#### Input Configuration
```
fields:
  - id: appkey
    type: string
    label: Twitter Application Key
  - id: appsecret
    type: string
    label: Twitter Application Secret
    secret: true
  - id: tokenkey
    type: string
    label: Twitter Token Key
  - id: tokensecret
    type: string
    label: Twitter Token Secret
    secret: true
required:
  - appkey
  - appsecret
  - tokenkey
  - tokensecret
```
#### Injector Configuration
```
extra_vars:
  app_key: '{{ appkey }}'
  app_secret: '{{ appsecret }}'
  token_key: '{{ tokenkey }}'
  token_secret: '{{ tokensecret }}'
```
</details>

#### SMTP
<details>
  <summary markdown="span">SMTP Credentials</summary>

#### Input Configuration
```
fields:
  - id: mailhost
    type: string
    label: Mail Host
  - id: mailport
    type: string
    label: SMTP Port
  - id: mailuser
    type: string
    label: Send Mail User
  - id: mailpassword
    type: string
    label: SMTP Mail Password
    secret: true
required:
  - mailhost
  - mailport
  - mailuser
  - mailpassword

```
#### Injector Configuration
```
extra_vars:
  mail_host: '{{ mailhost }}'
  mail_password: '{{ mailpassword }}'
  mail_port: '{{ mailport }}'
  mail_user: '{{ mailuser }}'
```
</details>

### Google Cloud
This demo runs on Google Cloud, inside it's own project. The requirements for this GCP project are:
 * a valid Serive Account JSON for the GCP Project
 * a dynamic Inventory for that Project
 * a SSH keypair for the user *rhepdstower*, with the private Key stored inside the Tower as *Machine Credential* and the public key stored inside the Google Cloud Project under *Metadata* in the Section *ssh-keys*
 * a Network Target on Google Cloud Networking named *http-server* that points to the Google Cloud Firewall ingress Role *default-allow-http*.

## Inventories
This Demo needs two inventories
 * A dynmaic Inventory for the Google Cloud Project
 * A *local* inventory with one static Host-entry *localhost* and the Variable " `ansible_connection: local` " assigned to it

## Job Templates
### 1. VM Rollout
The VM-Rollout Template uses the Google Cloud Inventory, references the `loop.yml` Playbook and requires the extra variables: 
<details>
  <summary markdown="span"> Extra Variables</summary>
  
```
gce_type: n1-standard-2
gce_zone: us-central1-a
gce_source: projects/centos-cloud/global/images/family/centos-8
gce_disksize: 50
gce_machines:
  - elastic
  - logstash
gce_network_tags: http-server
```
</details>

### 2. Wait for VMs
Wait for VMs does not require extra vars, runs on the Google Cloud Inventory and references the Playbook `wait_gce.yml`

### 3. ELK-Stack Rollout
The rollout Template refers to the playbook `prep.yml` and requires the Credentials:
 * Twitter-API-Key
 * DDNS-Credentials
 * SSH-Machine Key of the *rhepdstower* User.

In addition, the following variables must be defined:
 * `tw_keyword` must contain the Keyword you want to index on twitter
 *  `nginx_user` and `nginx_password` must contain a Username/Password Combination that will be used by Nginx to authenticate the User to the rolled out Application.

You can define these Variables as *Extra Variables* in the Tempalte Definition, or use a survey.
The Demoy preferably uses a Workflow Template with the survey to query all necessary variables, so that the individual Job Template does not need to define them.

### 4. (optional) delete VMs
The delete-Template references the `loop_del.yml` Plabook and requires the same Variables as the VM-Rollout. This Template is optional only if you want to create a Workflow Template with *on failure* branches. 

## Workflow Template
Define a Workflow Template that queries the following Variables through a survey:
 * `tw_keyword`
 * `nginx_user`
 * `nginx_password`
 * `mail_to` will be used by the Sendmail-Playbook to send a final E-Mail about the completion of the Workflow

Define the Workflow with the folowwing steps:

1. Sync the Projects Git Repository
2. Roll Out the VMs
3. Sync the dynamic Inventory
4. Wait for the Machines
5. Roll Out the Application
6. Send an E-Mail 

The steps 2, 4 and 5 of the Workflow should have a branch *on failure*, pointing to the *delete VM* template which will remove the VMs from the Cloud environment.

*sample Screenshots from Tower and GCP will follow*

## FAQ:
 * *Can I run this on another cloud or Virtualization Plattform*
   * Sure. Just write the VM-Rollout Playbook for that. Keep in mind, that some of the Roles & Templates reference facts from Google Cloud, like `gce_private_ip`. You'll need to modify these Roles accordingly
 * *Can I use a different DDNS-Provider*
   * Sure, just write you own custom Role for that
 * *Do I need all Templates/Playbooks for a functioning Demo*
   * No, it works without DDNS and the Mail Send
 * *Why do you roll out 2 VMs, whereas the ELK-Stack would work on only one*
   * This Demos should showcase the autmated rollout of an application that runs multiple VMs. There's no "functional" reason to separate Logstash from Elastic. The first Version of this Demo even deployed 5 VMs: Logstash, Kibana and three Elastic Data Nodes in a Cluster. But that one simply took to long to rollout.

## Work in Progress
 * Write the Application-Rollout Playbook agnostic from the Cloud Roll Out
 * add multiple Clouds as target with a Survey to choose from (Do you want to run on AWX, GCP, IBM, local RHV ? ... etc)
 * add a *Post rollout* Playbook that automatically creates the Kibana Index, some Views and a Dashboard


