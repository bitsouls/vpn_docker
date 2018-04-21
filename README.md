# OpenVPN ipv6 server docker

## Setting up an ipv6 capable Amazon instance
1. Obtain an Amazon account.
2. Select the preferred region where you want your VPN server to reside. EU-Central (Frankfurt) for example. Go to
   Services > VPC. Select the default VPC there. Actions > Edit CIDRs. Add IPV6 CIDR (leave blank for Amazon to
   auto-assign an IPV6 CIDR).
3. Go back to VPC main page. Select Subnets. There should be 3 default subnets. Select the first subnet.
   Subnet Actions > Edit IPV6 CIDRs. Leave the automatically suggested CIDR there and apply it.
   Subnet Actions > Modify auto-assign IP settings. Ensure both checboxes are checked and hit save. Close the dialog box
   and repeat the process for every other subnet.
4. Navigate to route tables. Select the default route table there. Subnet Associations > Edit, select all of the subnets
   and save. Go to Routes tab, hit Edit, hit Add another route. Input ::/0 as destination and pick the default internet
   gateway as target. Hit Save.
5. Navigate to Security Groups. Hit Create Security Group. Give it some name and description. Select your new Security
   Group and open the Inbound Rules tab. Hit Edit. You should add the following rules:
   ```
   Custom UDP Rule  -  UDP (17)  -  1194  -  0.0.0.0/0
   Custom UDP Rule  -  UDP (17)  -  1194  -  ::/0
   SSH (22)  -  TCP (6)  -  22  -  0.0.0.0/0
   SSH (22)  -  TCP (6)  -  22  -  ::/0
   Custom TCP Rule  -  TCP (6)  -  1194  -  0.0.0.0/0
   Custom TCP Rule  -  TCP (6)  -  1194  -  ::/0
   ```
   Hit Save.
6. Go to Services > EC2. Launch Instance. Select the Ubuntu 16 image. Select t2.micro and hit next. Select one of your
   subnets and ensure Auto-assign IPv6 IP is enabled (it should be by default which we've set up previously), hit Next.
   Skip the next steps until you get to "Configure Security Group". Check the Select an existing security group combo
   box and select the group we've set up in the previous step. Hit Review and launch and hit Launch. When prompted for
   security keys pick Create new Pair. Enter a name for them. Download the generated keys. You now have 1 running
   instance. If you don't know how to SSH into it, select the instance and hit connect for instructions. If it isn't
   accessible by its hostname or ip, find its ipv6 and use it instead of those.

## Installing docker and docker-compose
1. SSH into your running instance.
2. Run the following commands:
   ```
   sudo apt-get update
    
   sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
    
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    
   sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
    
   sudo apt-get update
    
   sudo apt-get install docker-ce
   ```
   ```
   sudo curl -L https://github.com/docker/compose/releases/download/1.21.0/docker-compose-$(uname -s)-$(uname -m) \
   -o /usr/local/bin/docker-compose
    
   sudo chmod +x /usr/local/bin/docker-compose
   ```
### Enabling ipv6 support for docker and OpenVPN
1. Run the following commands:
   ```
   sudo sed -e 's:^\(ExecStart.*\):\1 --ipv6:' /lib/systemd/system/docker.service | \
   tee /etc/systemd/system/docker.service
    
   sudo systemctl restart docker.service
    
   sudo curl -o /etc/systemd/system/docker-openvpn@.service \
   'https://raw.githubusercontent.com/kylemanna/docker-openvpn/dev/init/docker-openvpn%40.service'
   ```
2. You need to open the systemd Unit File we've downloaded in the previous step and edit it. Run
   ```
   sudo vi /etc/systemd/system/docker-openvpn@.service
   ``` 
   Find a commented out line that looks like Environment="IP6_PREFIX=" uncomment it and change it so IP6_PREFIX has
   the /64 prefix of the Amazon subnet of your running instance (you can look them up in Services > VPC > Subnets).
   Save the changes and run
   ```
   sudo systemctl daemon-reload
   ``` 
## Running OpenVPN
1. Clone this repo (git clone git://github.com/bitsouls/vpn_docker.git). Navigate into the repo directory
   (cd vpn_docker).
2. Configure OpenVPN server's address. Replace "udp://VPN.SERVERNAME.COM" with your instance public address and run
   ```
   docker-compose run --rm openvpn ovpn_genconfig -u udp://VPN.SERVERNAME.COM
   ```
3. Configure secret keys. Run
   ```
   docker-compose run --rm openvpn ovpn_initpki
   ```
4. Run the following commands:
   ```
   sudo chown -R $(whoami): ./openvpn-data
   docker-compose up -d openvpn
   ```
5. Generate client configs with
   ```
   export CLIENTNAME="your_client_name"
   docker-compose run --rm openvpn easyrsa build-client-full $CLIENTNAME (with passphrase key protection)
   docker-compose run --rm openvpn easyrsa build-client-full $CLIENTNAME nopass (without passphrase key protection)
   ```
   This should generate a file that will have your client name and "ovpn" extension, which you can use to connect to
   your OpenVPN server. View the contents of that file with
   ```
   cat $CLIENTNAME.ovpn
   ```
## What's next ?
Now you should install and run an OpenVPN client for your specific OS and connect to it using the client config file
we made in the previous section.
A few notes re client configs:
1. If you used the passphrase protection and can't input the passpharase upon connecting to your OpenVPN server
   (your client runs on your WiFi router for example) you should add the following line into your client config:
   ```
   askpass your_passphrase
   ```
2. You may get errors with "block-outside-dns" string in it. Add
   ```
   pull-filter ignore "block-outside-dns"
   ```
   to your config to get rid of it. 
