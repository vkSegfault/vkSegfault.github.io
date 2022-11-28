---
title: Hosting Godot 4 server on AWS and Azure
categories: [cloud, godot, aws, azure, terraform]
tags: [godot, aws, azure, python, boto3, terraform]     # TAG names should always be lowercase
---

Godot is slowly[^1] getting to release of version 4 - probably biggest update to date. Most interesting from our perspective is rewritten networking part. While there is stable Godot 3 relase as well, basically everyone is watching out for new major version. Some gamedevelopers will jump on this bandwagon soon (and some of them probably did it already). **In this article we will take a look how to setup Godot 4 server in headless mode and host it on 2 most popular public cloud provides: AWS and Azure.**

> Note that Godot 4 is currently in beta, there are bugs, some things might change before final release. It's definitely not production-ready release - keep it in mind. Nonetheless it's worth giving a shot just for new shiny networking.

If you want take a shallow dive on what's new with networking in Godot 4 take a look at this series of official blogs: 
1. [On servers, RSETs and state updates](https://godotengine.org/article/multiplayer-changes-godot-4-0-report-1)
2. [RPC syntax, channels, ordering](https://godotengine.org/article/multiplayer-changes-godot-4-0-report-2)
3. [ENet wrappers, WebRTC](https://godotengine.org/article/multiplayer-changes-godot-4-0-report-3)
4. [Scene Replication](https://godotengine.org/article/multiplayer-changes-godot-4-0-report-4).

> Note that we will deploy Godot server on AWS and Azure in fastest possible way, it's not neccesrily safest or properly structured.

## Setup Headless Godot Server

With newly baked networking system for Godot 4 setting up networking is much more pleasant and intuitive than it was with Godot 3.
First we need to set some variables upfront:
```gdscript
var ADDRESS: String
const PORT = 12077
const MAX_PLAYERS = 8
var server: bool
```
Use any random port above `9999`.  The `server` variable will hold info whether we run it as pure hosting peer. We can use `OS.get_cmdline_args()` to get array of all passed args and if - in my case - it's `--server` we set this variable to `true`. Note that ideally this should be handled in `_enter_tree()` callback (at the beginning) and not `_ready()` (at the end of loading process).

> We use `server` flag to have control over whether it should be *server only* or *server as peer*, in latter case we can host game and act as player similiar to any other clients.

So now we are ready to spin up our server:
```gdscript
func _ready():
  spawn_server()

func spawn_server():
  var peer = ENetMultiplayerPeer.new()
  peer.create_server(PORT, MAX_PLAYERS)
  multiplayer.multiplayer_peer = peer
	
  peer.peer_connected.connect(_on_peer_connected)
``` 
We first create ENet peer instance and use it to create server. `multiplayer` is global singleton that works over all nodes (we can set multiplayer per scene as well if needed).

Now that we have working server we want to spawn new character everytime client joins and we achieve this with `_on_peer_connected()` function tied to `peer_connected` signal.

`peer_connected` signal takes 1 arg that we will make use of:
```gdscript
func _on_peer_connected(peer_id):
  var character = load("res://path/to/character.tscn")
  var char = character.instantiate()
  char.name = str(id)
  self.add_child(char)
```
The important part is assigning peer id to `.name`of scene (actually global node of this scene) - we will use it later to differniate between peers.

---

Our code for server is ready, we just need to add 1 more Node - **MultiplayerSpawner**. This is a great new addition which makes setup a lot faster than writing it by hand. So add MultiplayerSpawner as child to your main scene, set *Spawn Path* to location when Characters are suppose to spawn for every peer and in *Auto Spawn List* add path to Character scene itself. That's it! No need to write any code that spawns scenes for every peer.

## Setup Godot Client

> Server will run in headless mode but for clients we want to see actual game.

> Note that Server and Client code may be merged into 1 file for simplicity and just execute proper path based whether it's server (cmdline arg) or client (paste IP and click Join button).

On client side we need to be able to paste IP address that we want to connect to. We will use **LineEdit** node:
```gdscript
var lineedit_ip = LineEdit.new()
lineedit_ip.placeholder_text = "127.0.0.1"
lineedit_ip.expand_to_text_length = true
self.add_child(lineedit_ip)
```

To setup Godot client that will join our server we proceed almost exactly the same as with server setup:
```gdscript
var peer = ENetMultiplayerPeer.new()
ADDRESS = lineedit_ip.text
peer.create_client(ADDRESS, PORT)
multiplayer.multiplayer_peer = peer
```
Only difference is `.create_client()` instead of `_create_server()`. We use the address and port that are set up to this point.

---

Code above set client peer and now we need to make some preparations for actual Character scene. First thing is to add another great node provided by Godot 4 which is **MultiplayerSynchronizer**. Add it as child of root node and set *Root Path* to root node as well. Now the important part: click on MultiplayerSynchronizer node and on the bottom, alongside Debugger, Audio, etc tabs you should see new one: **Replication**. Click it and use `+` sign to add properties that should be synced across all connected peers: mainly position and rotation but it's up to you what are yours particular needs.

> Only properties added to MultiplayerSynchronizer will be replicated to other peers!

Last 2 things that we need to do is to set authority and check if current peer is this authority. In `_ready()` function add `$MultiplayerSynchronizer.set_multiplayer_authority(str(name).to_int())`. Remember that in server paragraph we passed peer id to `name` property? This is used here to distinguish peers and only execute physics updates like movement for itself.

Then in `_process()` we just check `if synchronizer.is_multiplayer_authority()` and then run all physics code inside.

### Export project to executable

> For simplicity sake we mix both server and client side codebase into one file but in more production-ready environment it would be advised to separate them.

We will export our project into Linux executables so we can later use cloud providers as hosting. Save everything and go to **Project --> Export**. Once there, add template for `Linux/X11` and click it. Tick *runnable*, set export path and for simplicity sake leave all other flags with default values.

> Note that ready binaries will work on your local network OOTB, as well just run 1 as server and others as clients.

> Now we will deploy it to 2 popular cloud provides and just for fun we will use different ways to achieve it.

## Deploy server to AWS

> **For AWS we will use boto3**

Binaries are just blob files with right permission so we will use S3 bucket to store it. We will use popular `pipenv` tool to install `boto3`:
```bash
pipenv --three
pipenv install boto3 awscli
```
For simplicity we will use `pipenv run aws configure` to setup credentials.

#### S3
Now we need to create Bucket and upload our server there:
```python
s3 = boto3.client('s3', region_name="eu-central-1")

bucket_name = "godot-server-totally-unique-name-987654321"
file = "/path/to/SuperGodotKart"

try:
  location = {'LocationConstraint': "eu-central-1"}
  s3.create_bucket(Bucket=bucket_name, CreateBucketConfiguration=location)

  s3.upload_file(file, bucket_name, "SuperGodotKart.x86_64")
except ClientError as e:
  print(e)
```

#### IAM Policy and Role

Let's now create IAM role which will allow us access S3 bucket from EC2 instance. We will provide assume policy for Role creation first, so that EC2 will be able to call other services on our behalf:
```json
assume_policy_dict = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com"
                ]
            }
        }
    ]
}
```

and new explicit policy for what Principal can actually do:
```json
policy_dict = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::godot-server-totally-unique-name-987654321"
        }
    ]
}
```

Let's now actually create policy, role, instance profile and attach them all properly:
```python
try:
    policy = iam.create_policy(
        PolicyName = 'godot-policy',
        PolicyDocument = json.dumps(policy_dict)
    )
    policy_arn = policy['Policy']['Arn']

    role = iam.create_role(
        RoleName='godot-role',
        AssumeRolePolicyDocument=str(json.dumps(assume_policy_dict))
    )

    iam.attach_role_policy(
        RoleName='godot-role',
        PolicyArn=policy_arn
    )

    instance_profile = iam.create_instance_profile(InstanceProfileName='godot-instance-profile')
    instance_profile_arn = instance_profile['InstanceProfile']['Arn']   # used later
    iam.add_role_to_instance_profile(InstanceProfileName='godot-instance-profile', RoleName='godot-role')
except ClientError as e:
     print(e)
```

#### EC2

Now we are almost ready to deploy EC2 with all setup above. Last thing to do is to create Security Group that will allow ingress for UDP and TCP on ports 12077-12087 (Godot will use port provided at the beginning of this article and may use increments of this). Reason for this is that [default SG group created with EC2 allows all egress traffic but (fortunatelly) ingress from network interfaces that are using same SG only](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/default-custom-security-groups.html). Finally we provide *user data* to copy server from S3, install dependencies and actually run server:
```python
ec2 = boto3.client('ec2', region_name="eu-central-1")
user_data = """sudo apt install -y libxcursor-dev libxinerama-dev libxrandr-dev libxi-dev &
            aws s3 cp s3://godot-server-totally-unique-name-987654321/GodotServer . &
            chmod +x GodotServer &
            ./GodotServer --server --headless"""

try:
    sg = ec2.create_security_group(
        Description='Allow ingress for UDP/TCP on ports 12077-12087',
        GroupName='godot-sg'
    )
    sg_id = sg['GroupId']

    ec2.authorize_security_group_ingress(
        GroupId=sg_id,
        IpProtocol='udp',
        CidrIp='0.0.0.0/0',
        FromPort=12077,
        ToPort=12087
    )

    ec2.authorize_security_group_ingress(
        GroupId=sg_id,
        IpProtocol='udp',
        CidrIp='0.0.0.0/0',
        FromPort=12077,
        ToPort=12087
    )

    instances = ec2.run_instances(
        ImageId = "ami-0caef02b518350c8b", # Ubu 22.04 LTS
        MinCount=1,
        MaxCount=1,
        InstanceType="t2.micro",
        IamInstanceProfile={
            'Arn': instance_profile_arn,
            'Name': 'godot-instance-profile'
        },
        SecurityGroupIds=['sg_id'],
        NetworkInterfaces=[{'AssociatePublicIpAddress': True}],
        UserData=user_data
    )

except ClientError as e:
    print(e)
```

> Ami provided above is for Ubuntu 22.04 LTS and that's why OS-specific commands are for **apt**, if you choose to use Amazon Linux or RHEL please use `sudo yum install libXcursor libXinerama libXrandr libXi` instead.

> We used t2.micro just for this simple example. In production we should use beefy VMs (ideally with HVM support) like bigger T or even better: C family.

What we are actually doing above: spawning 1 instance of Ubuntu 22.04 running on t2.micro and then run Godot server that should be reachable by any clients.


## Deploy server to Azure

> **For Azure we will use Terraform**

First we should tackle case of AZ credentials without checking it in into any repository. For this example we will proceed with `.tfvars` file that is also added to `.gitignore`.

#### Access Azure from Terraform:
```terraform
# vars.tfvars
subscription_id = "<YOUR_SUB_ID>"
tenant_id = "<YOUR_TENANT_ID>"
client_id = "<YOUR_CLIENT_ID>"
client_secret = "<YOUR_CLIENT_SECRET>"
```

```tf
# vars.tf
variable "subscription_id" {
  type = string
  sensitive = true
}

variable "tenant_id" {
  type = string
  sensitive = true
}

variable "client_id" {
  type = string
  sensitive = true
}

variable "client_secret" {
  type = string
  sensitive = true
}
```

#### Provider:
```terraform
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0.2"
    }
  }

  required_version = ">= 1.1.0"
}

provider "azurerm" {
    features {}
    subscription_id = var.subscription_id
    tenant_id = var.tenant_id
    client_id = var.client_id
    client_secret = var.client_secret
}
```

> You can embed subscription, tenant, client ids and secret inside `provider` block for this example but it's strongly ill-advised.

#### Resource Group

We will use *West Europe* Region (Netherlands) for our Resource Group:
```terraform
resource "azurerm_resource_group" "az-vm-rg" {
    name = "az-vm-rg"
    location = "westeurope"
}
```

#### Blob Storage

Now let's upload Godot server to Blob Storage:
```terraform
resource "azurerm_storage_account" "storageaccount" {
    name = "uniquegodotstorage9876"
    resource_group_name = azurerm_resource_group.az-vm-rg.name
    location = azurerm_resource_group.az-vm-rg.location
    account_tier = "Standard"
    account_replication_type = "LRS"
}

resource "azurerm_storage_container" "container" {
    name = "container"
    storage_account_name = azurerm_storage_account.storageaccount.name
    container_access_type = "private"
}

resource "azurerm_storage_blob" "blob" {
    name = "GodotServer"
    storage_account_name = azurerm_storage_account.storageaccount.name
    storage_container_name = azurerm_storage_container.container.name
    type = "Block"
    source = "path/to/GodotServer"
}
```
 `container_access_type` is by default `private` but it's worth pointing it out.

#### Network Security Group Rules

Next step is to setup networking. It consist of 2 different parts: network security group rules and virtual network itself. Let's first set firewall rules to allow Godot server speak over UDP and TCP and allow ssh-ing into our machine (used for `remote-exec` later on): 
```terraform
locals {
  nsg_rules = {

    tcp_in = {
        name                       = "tcp_in"
        priority                   = 1001
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "12077-12087"
        destination_port_range     = "12077-12087"
        source_address_prefix      = "*"
        destination_address_prefix = "*"

    }

    udp_in = {
        name                       = "udp_in"
        priority                   = 1002
        direction                  = "Inbound"
        access                     = "Allow"
        protocol                   = "Udp"
        source_port_range          = "12077-12087"
        destination_port_range     = "12077-12087"
        source_address_prefix      = "*"
        destination_address_prefix = "*"

    }

    tcp_out = {
        name                       = "tcp_out"
        priority                   = 1003
        direction                  = "Outbound"
        access                     = "Allow"
        protocol                   = "Tcp"
        source_port_range          = "12077-12087"
        destination_port_range     = "12077-12087"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    udp_out = {
        name                       = "udp_out"
        priority                   = 1004
        direction                  = "Outbound"
        access                     = "Allow"
        protocol                   = "Udp"
        source_port_range          = "12077-12087"
        destination_port_range     = "12077-12087"
        source_address_prefix      = "*"
        destination_address_prefix = "*"
    }

    ssh_in = {
        name                       = "ssh_in"
        priority                   = 100
        direction                   = "Inbound"
        access                      = "Allow"
        protocol                    = "Tcp"
        source_port_range           = "*"
        destination_port_range      = "22"
        source_address_prefix       = "*"
        destination_address_prefix  = "*"
    }
  }
}
```
We can squeeze up some security by limiting `source_address_prefix` to some adresses we know.

#### Virtual Network

Now we can set virtual network that will use rules above:
```terraform
resource "azurerm_virtual_network" "vnet" {
  name                = "vnet"
  address_space       = ["10.0.0.0/16"]
  resource_group_name = azurerm_resource_group.az-vm-rg.name
  location            = azurerm_resource_group.az-vm-rg.location
}

resource "azurerm_subnet" "sub1" {
  name                 = "sub1"
  resource_group_name  = azurerm_resource_group.az-vm-rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_public_ip" "pubip1"   { 
   name                = "pubip1" 
   location            = azurerm_resource_group.az-vm-rg.location 
   resource_group_name = azurerm_resource_group.az-vm-rg.name 
   allocation_method   = "Static"
 } 

resource "azurerm_network_security_group" "nsg" {
  name                = "nsg"
  location            = azurerm_resource_group.az-vm-rg.location
  resource_group_name = azurerm_resource_group.az-vm-rg.name
}

resource "azurerm_network_security_rule" "nsg_rules" {
  for_each                    = local.nsg_rules
  name                        = each.key
  priority                    = each.value.priority
  direction                   = each.value.direction
  access                      = each.value.access
  protocol                    = each.value.protocol
  source_port_range           = each.value.source_port_range
  destination_port_range      = each.value.destination_port_range
  source_address_prefix       = each.value.source_address_prefix
  destination_address_prefix  = each.value.destination_address_prefix
  resource_group_name         = azurerm_resource_group.az-vm-rg.name
  network_security_group_name = azurerm_network_security_group.nsg.name
}

resource "azurerm_network_interface" "nic" {
  name                = "nic"
  location            = azurerm_resource_group.az-vm-rg.location
  resource_group_name = azurerm_resource_group.az-vm-rg.name

  ip_configuration {
    name                          = "ipconfig"
    subnet_id                     = azurerm_subnet.sub1.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pubip1.id
  }
}

# Connect the security group to the network interface
resource "azurerm_network_interface_security_group_association" "example" {
  network_interface_id      = azurerm_network_interface.nic.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}
```
 **In `azurerm_public_ip` block we may use `Dynamic` allocation, but then we must use `data` block as well to fetch it.**

#### Virtual Machine

The last thing we need to do is: spin up 1 VM of Ubuntu 22.04 and remotely exec installation of all dependencies, fetch Godot server binary from Azure Blob and run it.

```terraform
resource "azurerm_linux_virtual_machine" "godot-vm" {
  name                  = "godot-vm"
  resource_group_name   = azurerm_resource_group.az-vm-rg.name
  location              = azurerm_resource_group.az-vm-rg.location
  network_interface_ids = [azurerm_network_interface.nic.id]
  size                  = "Standard_B1s"
  admin_username        = "crazyadmin"
  admin_password        = "Dinozaury1"
  
  # we can use separate resource "azurerm_ssh_public_key" instead
  admin_ssh_key {   
    username            = "crazyadmin"
    public_key          = "${file("keys/key.pub")}"    
  }
  disable_password_authentication = false

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  os_disk { 
    caching              = "ReadWrite" 
    storage_account_type = "Standard_LRS" 
  }


    connection {
      type        = "ssh"
      host        = azurerm_public_ip.pubip1.ip_address
      user        = azurerm_linux_virtual_machine.godot-vm.admin_username
      password    = azurerm_linux_virtual_machine.godot-vm.admin_password
      private_key = "${file("keys/key")}"
      timeout     = "5m"
    }

    provisioner "remote-exec" {
        inline = [
            "sudo apt install -y libxcursor-dev libxinerama-dev libxrandr-dev libxi-dev",
            "curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash",
            "az storage blob download --account-name ${azurerm_storage_account.storageaccount.name} -c ${azurerm_storage_container.container.name} -n ${azurerm_storage_blob.blob.name} --sas-token '${data.azurerm_storage_account_sas.sas_token.sas}'",
            "./Godot --server --headless"
        ]
    }
}
```

> Instead of `az` CLI utility we can use handy `azcopy`: 
> `wget https://aka.ms/downloadazcopy-v10-linux`
> `tar -xvf downloadazcopy-v10-linux && cd azcopy_linux_amd64_10.16.2/`
> `./azcopy copy 'https://uniquegodotstorage9876.blob.core.windows.net/container/Godot?<YOUR_SAS_TOKEN>' '.'`

Now just call: 
1. `terraform init`
2. `terraform plan -var-file=vars.tfstate -out=out.plan` 
3.  and finally `terraform apply out.plan`

As you can see, logic here is more or less similar to what we did with AWS and Boto3 earlier.

> Note that you should always use versioning and resource-locking of your Terraform `state`!


## Security measures

Note that this post doesn't cover security aspects that should be normally our top priority. Main purpose of this post is to make simplest possible server-client architecture and deploy it. For production there is a lot more we would need to set before going public. 

## Footnotes

[^1]: https://en.wikipedia.org/wiki/Waiting_for_Godot
