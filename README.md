[Nagios Installtion](#nagios-installation)

[Installing the check_nrpe Plugin](#installing-the-check-nrpe-plugin)

# Nagios Installation

Create a nagios user and nagcmd group. You’ll use these to run the Nagios process.

    useradd nagios
    groupadd nagcmd

Then add the user to the group:

    usermod -a -G nagcmd nagios
    
Update your package and install the required packages:

    apt-get update
    apt-get install build-essential libssl-dev unzip
    
Download Nagios using `curl` command and extract:
    
    cd ~ 
    apt install curl && curl -L -O https://assets.nagios.com/downloads/nagioscore/releases/nagios-4.4.4.tar.gz
    tar zxf nagios-*.tar.gz
    cd nagios-4.4.4 

Run the `configure` script to specify the user and group you want Nagios to use. Use the `nagios` user and `nagcmd` group you created:

    ./configure --with-nagios-group=nagios --with-command-group=nagcmd

```If you want Nagios to send emails using Postfix, you must install Postfix and configure Nagios to use it by adding "--with-mail=/usr/sbin/sendmail" to the configure command. We won’t cover Postfix in this tutorial, but if you choose to use Postfix and Nagios later, you’ll need to reconfigure and reinstall Nagios to use Postfix support.```

Now compile Nagios with this command and configure:

    make all
    make install
    make install-commandmode
    make install-init
    make install-config

You’ll use Apache to serve Nagios’ web interface, so copy the sample Apache configuration file to the `/etc/apache2/sites-available` folder:

    /usr/bin/install -c -m 644 sample-config/httpd.conf /etc/apache2/sites-available/nagios.conf
    usermod -G nagcmd www-data

Nagios is now installed. Let’s install a plugin which will allow Nagios to collect data from various hosts.

## Installing the check-nrpe Plugin

    cd ~
    curl -L -O https://github.com/NagiosEnterprises/nrpe/releases/download/nrpe-3.2.1/nrpe-3.2.1.tar.gz
    tar zxf nrpe-3.2.1.tar.gz
    cd nrpe-3.2.1
    ./configure
    make check_nrpe
    make install-plugin

## Configuring Nagios

Open the main Nagios configuration file in your text editor:

    sudo nano /usr/local/nagios/etc/nagios.cfg

Find this line in the file:

    ....
    #cfg_dir=/usr/local/nagios/etc/servers
    ....

Uncomment this line by deleting the # character from the front of the line:

    cfg_dir=/usr/local/nagios/etc/servers

Save the file and exit the editor.

Now create the directory that will store the configuration file for each server that you will monitor:

    sudo mkdir /usr/local/nagios/etc/servers

Open the Nagios contacts configuration in your text editor:

    sudo nano /usr/local/nagios/etc/objects/contacts.cfg


Find the `email` directive and replace its value with your own email address.

Next, add a new command to your Nagios configuration that lets you use the check_nrpe command in Nagios service definitions.

    sudo nano /usr/local/nagios/etc/objects/commands.cfg

Add the following to the end of the file to define a new command called check_nrpe:

    …
    define command{
        command_name check_nrpe
        command_line $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
    }

Save and exit the editor.

Now configure Apache to serve the Nagios user interface. Enable the Apache rewrite and cgi modules with the a2enmod command:

    sudo a2enmod rewrite
    sudo a2enmod cgi

Use the htpasswd command to create an admin user called nagiosadmin that can access the Nagios web interface:

    sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

Enter a password at the prompt. Remember this password, as you will need it to access the Nagios web interface.

**Note**: If you create a user with a name other than nagiosadmin, you will need to edit `/usr/local/nagios/etc/cgi.cfg` and change all the `nagiosadmin` references to the user you created.

Now create a symbolic link for nagios.conf to the sites-enabled directory. This enables the Nagios virtual host.

    sudo ln -s /etc/apache2/sites-available/nagios.conf /etc/apache2/sites-enabled/

Next, open the Apache configuration file for Nagios.

    vi /etc/apache2/sites-available/nagios.conf

If you’ve configured Apache to serve pages over HTTPS, locate both occurrences of this line:

    #  SSLRequireSSL

Uncomment both occurrances by removing the # symbol.

If you want to restrict the IP addresses that can access the Nagios web interface so that only certain IP addresses can access the interface, find the following two lines:

    Order allow,deny
    Allow from all

Comment them out by adding # symbols in front of them:

    # Order allow,deny
    # Allow from all

Then find the following lines:

    #  Order deny,allow
    #  Deny from all
    #  Allow from 127.0.0.1

Uncomment them by deleting the # symbols, and add the IP addresses or ranges (space delimited) that you want to allow to in the Allow from line:

    Order deny,allow
    Deny from all
    Allow from 127.0.0.1 your_ip_address

These lines appear twice in the configuration file, so ensure you change both occurrences. Then save and exit the editor.

Restart Apache to load the new Apache configuration:


    sudo service nagios restart
    sudo nano /etc/systemd/system/nagios.service

Enter the following definition into the file

    [Unit]
    Description=Nagios
    BindTo=network.target

    [Install]
    WantedBy=multi-user.target

    [Service]
    Type=simple
    User=nagios
    Group=nagios
    ExecStart=/usr/local/nagios/bin/nagios /usr/local/nagios/etc/nagios.cfg

Save the file and exit your editor.

Then start Nagios and enable it to start when the server boots:

    sudo systemctl enable /etc/systemd/system/nagios.service
    sudo systemctl start nagios

Nagios is now running, so let’s log in to its web interface.

**Note**: If not working all Nagios plugin. run following command:

    apt-get install nagios-plugins

Copy plugin according your requirement:

    cp /usr/lib/nagios/plugins/plugin_name /usr/local/nagios/libexec/

## Installing NPRE on a Host

Log in to the second server, which we’ll call the **monitored server**.

    apt update
    apt install -y nagios-nrpe-server nagios-plugins

Modify the NRPE configuration file to accept the connection from the Nagios server, Edit the `/etc/nagios/nrpe.cfg` file.

    vi /etc/nagios/nrpe.cfg

Add the Nagios servers IP address, separated by comma like below.

    allowed_hosts=192.168.1.10

The `/etc/nagios/nrpe.cfg` file contains the basic commands to check the attributes (CPU, Memory, Disk, etc.architecure) and services (HTTP, FTP, etc.) on remote hosts. Below command lines let you monitor attributes with the help of Nagios plugins.

    command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
    command[check_load]=/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
    command[check_root]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /dev/mapper/server--vg-root
    command[check_swap]=/usr/lib/nagios/plugins/check_swap -w 20% -c 10%
    command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200

`In the above command definition` **`-w`** `stands for warning and` **`-c`** `stands for critical.`

## Add a Linux host to Nagios server

Create a client configuration file `/usr/local/nagios/etc/servers/host.local.cfg` to define the host and service definitions of remote Linux host.

    vi /usr/local/nagios/etc/servers/host.local.cfg

Copy the below content to the above file.

You can also use the following template and modify it according to your requirement. The following template is for monitoring logged in users, system load, disk usage (/ – partitions), swap, and total process.

    define host{
                           
            use                     linux-server            
            host_name               client.itzgeek.local            
            alias                   client.itzgeek.local            
            address                 192.168.1.20                     
    }                                   
                                    
    define hostgroup{                   
                                    
            hostgroup_name          linux-server            
            alias                   Linux Servers            
            members                 client.itzgeek.local
    }                                   
                                    
    define service{                     
                                    
            use                     local-service            
            host_name               client.itzgeek.local            
            service_description     SWAP Uasge            
            check_command           check_nrpe!check_swap                                                          
    }                                   
                                    
    define service{                     
                                    
            use                     local-service            
            host_name               client.itzgeek.local            
            service_description     Root / Partition            
            check_command           check_nrpe!check_root                                                          
    }                                   
                                    
    define service{                     
                                    
            use                     local-service            
            host_name               client.itzgeek.local            
            service_description     Current Users            
            check_command           check_nrpe!check_users                         
    }                                   
                                    
    define service{                     
                                    
            use                     local-service            
            host_name               client.itzgeek.local            
            service_description     Total Processes            
            check_command           check_nrpe!check_total_procs                                                   
    }                                   
                                    
    define service{                     
                                    
            use                     local-service            
            host_name               client.itzgeek.local            
            service_description     Current Load            
            check_command           check_nrpe!check_load
    }

Verify Nagios for any errors.

    /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

Restart the Nagios server.

    systemctl restart nagios

Check NRPE:

    /usr/local/nagios/libexec/check_nrpe -H localhost


# Install Postfix

    sudo apt-get install postfix mailutils libsasl2-2 ca-certificates libsasl2-modules

while installing Postfix will ask about some settings:

- choose `Internet Site`
- choose domain name for unqualified mail addresses
allow networks should be kept at 127.0.0.1 if you want local relay or you can specify list of IPs or networks from which to allow relay

After installation is over go to Postfix’s config:

    sudo nano /etc/postfix/main.cf

and add this to the end:

    relayhost = [smtp.gmail.com]:587
    smtp_sasl_auth_enable = yes
    smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
    smtp_sasl_security_options = noanonymous
    smtp_tls_CAfile = /etc/postfix/cacert.pem
    smtp_use_tls = yes

Now we need to define username and password for gmail, edit this file:

sudo nano /etc/postfix/sasl/sasl_passwd

And add the following line:

    smtp.gmail.com    USERNAME@gmail.com:PASSWORD

Once you have entered your username / password run postmap to create postfix lookup table:

    postmap /etc/postfix/sasl/sasl_passwd

This will create `sasl_passd.db` lookup table in same directory.

Next make sure that `sasl_passwd` and `sasl_passd.db` files have the right permissions:

    chown -R root:postfix /etc/postfix/sasl
    chmod 750 /etc/postfix/sasl
    chmod 640 /etc/postfix/sasl/sasl_passwd*

Restart postfix

    sudo service postfix restart

And test if email relays are working:

    echo "Tested" | mail -s "Testing Gmail Relay" user@example.net

## Nagios setup

First define contacts that you wish to notify and their email addresses in:

    sudo nano /usr/local/nagios/etc/objects/contacts.cfg

In order to change `sendmail` to `mailx` or `mail` for example we need to edit

    sudo nano /usr/local/nagios/etc/objects/commands.cfg

and change notify-host-by-email and notify-service-by-email commands:

    # 'notify-host-by-email' command definition
    define command{
	command_name	notify-host-by-email
	command_line	/usr/bin/printf "%b" "***** Nagios *****\n\nNotification Type: $NOTIFICATIONTYPE$\nHost: $HOSTNAME$\nState: $HOSTSTATE$\nAddress: $HOSTADDRESS$\nInfo: $HOSTOUTPUT$\n\nDate/Time: $LONGDATETIME$\n" | /usr/bin/mailx -s "** $NOTIFICATIONTYPE$ Host Alert: $HOSTNAME$ is $HOSTSTATE$ **" $CONTACTEMAIL$
	}
	
    # 'notify-service-by-email' command definition
    define command{
	command_name	notify-service-by-email
	command_line	/usr/bin/printf "%b" "***** Nagios *****\n\nNotification Type: $NOTIFICATIONTYPE$\n\nService: $SERVICEDESC$\nHost: $HOSTALIAS$\nAddress: $HOSTADDRESS$\nState: $SERVICESTATE$\n\nDate/Time: $LONGDATETIME$\n\nAdditional Info:\n\n$SERVICEOUTPUT$\n" | /usr/bin/mailx -s "** $NOTIFICATIONTYPE$ Service Alert: $HOSTALIAS$/$SERVICEDESC$ is $SERVICESTATE$ **" $CONTACTEMAIL$
	}

## Configure Critical and Warning mail id is different

First create two contact template:

    vi templates.cfg

Configure contact and add following line:
> **Note:** one contact add and another is default

    define contact {

    name                            generic-contact
    service_notification_period     24x7           
    host_notification_period        24x7           
    service_notification_options    w,u,c,r,f,s   
    host_notification_options       d,u,r,f,s     
    service_notification_commands   notify-service-by-email 
    host_notification_commands      notify-host-by-email    
    register                        0   
    }

    define contact {

    name                            critical-contact 
    service_notification_period     24x7             
    host_notification_period        24x7             
    service_notification_options    u,c,r,f,s        
    host_notification_options       d,u,r,f,s        
    service_notification_commands   notify-service-by-email 
    host_notification_commands      notify-host-by-email    
    register                        0   
    }

Configure contact to contacts.cfg file:

    vi contacts.cfg 

> **Note:** one contact add and another is default

    define contact {

    contact_name            nagiosadmin     
    use                     generic-contact 
    alias                   Nagios Admin            ; Full name of user
    email                   sample@gmail.com ; <<**** CHANGE THIS TO YOUR EMAIL ADDRESS 
    }

    define contact {

    contact_name            test  
    use                     critical-contact 
    alias                   Linux Admin 
    email                   sample@study24x7.com ; <<**** CHANGE THIS TO YOUR EMAIL ADDRESS 
    }


    define contactgroup {

    contactgroup_name       admins
    alias                   Nagios Administrators
    members                 nagiosadmin,test
    }


# Create Custom plugin to Nagios

    cd /usr/local/nagios/libexec

Create bash file like:

    vi check_mongodb.sh

Add following line for mongodb:

    #!/bin/bash

    status=$(systemctl status mongod.service | grep Active | awk '{print $2}')
    UPTIME=$(systemctl status mongod.service | grep Active| awk '{$1=$2=$3=""; print $0}')

    if [ "$status" = "active" ]
    then
        echo "Uptime - $UPTIME"
    exit 0
    else
        echo "DIST Testing MongoDB service is not working , Please check.."
    exit 2
    fi

After that configure remote server or local server rules:

    cd /usr/local/nagios/etc/objects

If configure rule local server edit to `localhost.cfg` and if configure rules remote server, So edit remote host file

    vi localhost.cfg
               OR
    vi ../servers/Remote_Host_Name.cfg

Add following line and add according your configuration:

    define service {

    use                     local-service           ; service template to use
    host_name               LOCAL_SERVER
    service_description     DIST Testing MongoDB
    check_command           check_mongodb
    }

Configuration `commands.cfg` file

    vi commands.cfg

Add following line according your configration:

    define command{
        command_name check_mongodb
        command_line /usr/local/nagios/libexec/check_mongodb.sh
    }

Check all syntex is done:

> ***/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg***

Restart Nagios Server

    systemctl restart nagios.service

**Reference for some links:**

https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/objectdefinitions.html

https://www.itzgeek.com/how-tos/linux/centos-how-tos/monitor-remote-linux-system-with-nagios-3.html

https://www.howtoforge.com/tutorial/ubuntu-nagios/

https://www.digitalocean.com/community/tutorials/how-to-install-nagios-4-and-monitor-your-servers-on-ubuntu-16-04

https://www.sib.in.rs/2016/nagios-postfix-gmail/
