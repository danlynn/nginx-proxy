# Nginx Proxy/HTTP Server Setup

This shows you how to configure nginx running within a docker container to proxy subdomains to random external host:ports OR to ports listening on the docker host outside of the nginx container.

Proxying connections to ports on the docker host is useful because this allows you to do things like give your customer a link like `http://rails.mydomain.org/retailer/123/customers.html` to view your latest website change.  However, since your webapp is running on your development laptop which is not directly accessible because you work in an office behind a firewall, you need to make use of the nginx proxy to facilitate the connection.

## What it does

In simple terms, this allows you to have a computer at your house that is always ON be able to make available services running on your laptop behind a firewall at some other location.

## The components involved

In more detailed terms, you have the following network topology components:

 + Dynamic DNS provider

   This service will be dynamcially associating a domain (like: mydomain.org) with your house's external IP.  You may want to configure a wildcard subdomain so that all URLs (like: http://rails.mydomain.org or http://laptop.mydomain.org) will resolve to your house's external IP.

 + Computer at home

   This computer is behind a regular NAT router (like wifi) at your house.  You will need to configure your NAT router to forward ports 22 and 80 to your computer.  This computer will be running nginx in a docker container.

 + Development Laptop at work

   This laptop is behind a firewall and is hosting a service (like a rails app) that needs to be accessed externally via an URL like http://rail.mydomain.org.

## Configure dynamic DNS

This configuration will depend on your dynamic DNS service.  Basically, you want to add some domains to your domain provider which then points them to your dynamic DNS provider which then points them to your house's external IP.

## Computer at home: Configuration

1. Configure your home's router to forward ports 22 and 80 to your computer.

2. Clone this repo to your computer.

3. Install Docker on your computer if you don't already have it.

4. Configure ssh on your computer to allow non-localhost connections to be forwards through reverse ssh tunnels.

   1. Add/Change the GatewayPorts config in the `/private/etc/ssh/sshd_config` file to appear as follows:

      <pre>
      GatewayPorts yes
      </pre>

      You will need to use `sudo vi /private/etc/ssh/sshd_config` to edit the file since it is owned by root.

   2. Restart sshd

      If using linux:

      ```bash
      $ sudo service sshd restart
      ```

      If using MacOS:

      ```bash
      $ sudo launchctl stop com.openssh.sshd
      ```

5. Configure a permanent static IP to be associated with your computer's loopback address.  This will allow the docker container where nginx is running to access your computer regardless of arbitrary IP changes assigned by your router's DHCP.

   Note that it is easy to alias an IP address to your loopback via:

   ```bash
   $ sudo ifconfig lo0 alias 192.168.111.111
   ```

   However, this assignment will disappear after your next reboot.  In order to make this change permanent, we will actually just be setting up a launchd daemon to run this command upon every reboot of your laptop.  Note that you will need sudo permissions in order to perform this step (if you don't run with admin privileges). If you just setup your Mac with default user that was created when you first got your laptop then you will by default have admin privileges.  But, since many advanced users who might be following these instructions will have created a non-privileged user for their regular account, I've added a couple steps here to show you how to temporarily add then remove yourself from the list of sudoers - thus, temporarily granting yourself sudo (root) permissions.

   1. If you don't run with admin privileges then perform the following steps to    grant yourself sudo access:
      
      1. Switch to your admin account, enter your admin password then launch    visudo:
         
         ```bash
         $ su admin
         Password: *****
         $ sudo visudo
         Password: *****
         ```
         
         This will open the sudoers file in the 'vi' editor.  Do not attempt to    edit the sudoers file in any other way.
      
      2. Add your regular account username below the root and %admin group entries    like the following (except use your regular username instead of 'danlynn'):
      
         ```bash
         # root and users in group wheel can run anything on any machine as any    user
         root            ALL = (ALL) ALL
         %admin          ALL = (ALL) ALL
         danlynn         ALL = (ALL) ALL
         ```
         
         These lines are usually found near the bottom of the file.  After adding    that last line, simply save the changes and exit back to the command line.
      
   2. Create a shell script to execute the IP alias command from the previous    section:
      1. Create a `.login` directory in your home directory
      
         ```bash
         $ cd ~
         $ mkdir .login
         $ cd .login
         $ touch alias_loopback.sh
         $ open .
         ```
         
      2. That last command should open a Finder window displaying the new file.     Open that file in your favorite editor, add the following, then save it:
         
         ```bash
         #!/bin/sh
         ifconfig lo0 alias 192.168.111.111
         ```
   3. Create a launchd daemon plist file to execute the `alias_loopback.sh` file    upon startup of your laptop:
      1. Create the `alias_loopback.plist` daemon plist file and set its ownership    and permissions
         
         ```bash
         $ cd /Library/LaunchDaemons/
         $ sudo touch alias_loopback.plist
         Password: *****
         $ sudo chown root:wheel alias_loopback.plist
         $ sudo chmod 755 alias_loopback.plist
         ```
         
         Note that you will only have to enter your regular user permissions on the first use of sudo.  The elevated sudo permissions will stay in effect for about 5 minutes before being rescinded.
         
      2. Open the `alias_loopback.plist` file in your favorite editor (which allows you to authenticate as admin to save changes - like sublime)
      3. Modify the contents of the `alias_loopback.plist` file as follows:
         
         ```xml
         <plist version="1.0">
             <dict>
                 <key>Label</key>
                 <string>alias_loopback</string>
                 <key>RunAtLoad</key>
                 <true />
                 <key>Program</key>
                 <string>/Users/danlynn/.login/alias_loopback.sh</string>
             </dict>
         </plist>
         ```
      4. In theory launchd should see when the file is saved and pick up and run it.  However, if it doesn't then try the following:
         
         ```bash
         $ sudo launchctl load /Library/LaunchDaemons/alias_loopback.plist
         ```
         
      5. Test that the launch control daemon is working by restarting your laptop and then pinging the new static IP alias:
         
         ```bash
         $ ping 192.168.111.111
         
         PING 192.168.111.111 (192.168.111.111): 56 data bytes
         64 bytes from 192.168.111.111: icmp_seq=0 ttl=64 time=0.053 ms
         64 bytes from 192.168.111.111: icmp_seq=1 ttl=64 time=0.045 ms
         64 bytes from 192.168.111.111: icmp_seq=2 ttl=64 time=0.050 ms
         64 bytes from 192.168.111.111: icmp_seq=3 ttl=64 time=0.034 ms
         ```
         
         If you get timeout messages instead of results like this then try going    back over these instructions and figuring out what you missed.  It might    be useful to tail the system log for launchd messages in this case:
         
         ```bash
         $ sudo tail -f /var/log/system.log
         ```
         
         Also, you can stop and restart the daemon without a reboot with:
         
         ```bash
         $ sudo launchctl unload /Library/LaunchDaemons/alias_loopback.plist
         $ sudo launchctl load /Library/LaunchDaemons/alias_loopback.plist
         ```
         
   4. Remove yourself from the sudoers file (if you don't run with admin    privileges)
      1. Edit the sudoers file:
            
         ```bash
         sudo visudo
         Password: *****
         ```
      2. Remove your regular username from the list of users with sudo privileges    so that it only contains the original root and admin entries like:
         
         ```
         # root and users in group wheel can run anything on any machine as any    user
         root            ALL = (ALL) ALL
         %admin          ALL = (ALL) ALL
         ```

6. Customize the `config/servers/conf` nginx config file to match the subdomain that you want to redirect to your laptop.  This file is located in this config directory in this project.  The example servers.conf file is setup to forward requests for `http://rails.mydomain.org/` to port 3000 on the static loopback IP that we setup for your computer `http://192.168.111.111:3000`:

   <pre>
   upstream rails {
       server 192.168.111.111:3000;
   }
   
   
   server {
           listen 80;
           server_name rails.mydomain.org;
   
           location / {
                   proxy_pass http://rails/;
                   proxy_set_header Host $host;
           }
   }
   </pre>

   The "upstream" server defines the destination host named "rails".  The `proxy_pass` statement specifies that the requests should be proxied to the upstream host named "rails".

   The `server` section specifies that requests arriving on port 80 to the `server_name` "rails.mydomain.org" will be processed.  

   The `location /` specifies that any path following the root will be carried over to the proxied upstream host URL.  Thus, "http://rails.mydomain.org/" => "http://192.168.111.111:3000/" and "http://rails.mydomain.org/users/5.html" => "http://192.168.111.111:3000/users/5.html".

   You may add additional `upstream` and `server` entries into this file to create additional proxies.

   This file is shared with nginx when the docker container is started.  So, if you make changes, you will need to restart the docker container via:

   ```bash
   $ docker-compose restart
   ```

7. Start the nginx docker container:

   ```bash
   $ docker-compose start
   ```

   You can display the logs with:

   ```bash
   $ docker-compose logs
   ```

   If you want to tail the logs then add an `-f` option on the end of that command.

   The `docker-compose.yml` specifies that this 'web' service should always be launched upon a restart of the computer.  Therefore, you won't need to worry about restarting the nginx container if you need to restart your computer for an OS update or whatever.

   Note that any http requests sent to this computer using a hostname that is not being proxied will simply be handled directly by nginx.  It is configured to host any content found in the `html` directory of this project.

## Share a service from your laptop

1. Start a service ON YOUR LAPTOP which you want to be proxied by the computer at your house.  For example, I'll be running a rails app on port 3000.

2. Open a reverse ssh tunnel from your laptop to the computer at your house.  Execute the following ON YOUR LAPTOP:

   ```bash
   $ ssh user@home.mydomain.org -R 0.0.0.0:3000:127.0.0.1:3000
   ```
   
   This command specifies that any computer (0.0.0.0) which attempts to connect to port 3000 on the computer at your house will have their connection tunneled back to your laptop to hit the rails server running on port 3000 (the second 3000 number).  We could have just as easily specified that it should connect to a tomcat server running locally on port 8080, etc.  The "127.0.0.1" refers to your laptop's loopback IP.  This is the same as "localhost".  However, in certain situations, "127.0.0.1" will work where "localhost" won't.  I use the IP just to be safe. 
