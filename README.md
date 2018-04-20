# OpenVPN ipv6 server docker

## Setting up an ipv6 capable Amazon instance
1. Obtain an Amazon account.
2. Go to Services > VPC. Select the default VPC there. Actions > Edit CIDRs. Add IPV6 CIDR (leave blank for Amazon to
   auto-assign an IPV6 CIDR).
3. Go back to VPC main page. Select Subnets. There should be 3 default subnets. Select the first subnet.
   Subnet Actions > Edit IPV6 CIDRs. Leave the automatically suggested CIDR there and apply it.
   Subnet Actions > Modify auto-assign IP settings. Ensure both checboxes are checked and hit save. Close the dialog box 
   and repeat the process for every other subnet.
4. Navigate to route tables. Select the default route table there. Subnet Associations > Edit, select all of the subnets
   and save. Go to Routes tab, hit Edit, hit Add another route. Input ::/0 as destination and pick the default internet gateway
   as target. Hit Save.
5. Navigate to Security Groups. Hit Create Security Group. Give it some name and description. Select your new Security Group 
   and open the Inbound Rules tab. Hit Edit. You should add the following rules:
       Custom UDP Rule  -  UDP (17)  -  1194  -  0.0.0.0/0
       Custom UDP Rule  -  UDP (17)  -  1194  -  ::/0
       SSH (22)  -  TCP (6)  -  22  -  0.0.0.0/0
       SSH (22)  -  TCP (6)  -  22  -  ::/0
       Custom TCP Rule  -  TCP (6)  -  1194  -  0.0.0.0/0
       Custom TCP Rule  -  TCP (6)  -  1194  -  ::/0
   Hit Save.
6. Go to Services > EC2. Launch Instance. Select the Ubuntu 16 image. Select t2.micro and hit next. Select one of your subnets
   and ensure Auto-assign IPv6 IP is enabled (it should be by default which we've set up previously), hit Next. Skip the next
   step until you get to "Configure Security Group". Check the Select an existing security group combo box and select the
   group we've set up in the previous step. Hit Review and launch and hit Launch. When prompted for security keys pick
   Create new Pair. Enter a name for them. Download the generated keys. You now have 1 running instance. If you don't know
   how to SSH into it, select the instance and hit connect for instructions. If it isn't accessible by its hostname or ip,
   find its ipv6 and use it instead of those.
