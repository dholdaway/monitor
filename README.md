# PHP/Bash server status monitoring

## Features
- Ping monitor
- History per host
- Threshold per monitored item.
- Monitors:
  - Processes (lighttpd, apache, nginx, sshd, munin etc.) 
  - RAM
  - Disk
  - Uptime
  - Users logged on
  - Updates
  - Network (RX/TX)

## Install

### Client

The client.sh script is a bash script which outputs JSON. It requires root access and should be run as root. It also requires a webserver, so that the server can get the json file. 

Software needed for the script:

- bash 
- awk 
- grep 
- ifconfig
- package managers supported: apt-get, yum and pacman (debian/ubuntu, centos/RHEL/SL, Arch)

Setup a webserver (lighttpd, apache, boa, thttpd, nginx) for the script output. If there is already a webserver running on the server you dont need to install another one.

Edit the script:

Network interfaces. First one is used for the IP, the second one is used for bandwith calculations. This is done because openVZ has the "venet0" interface for the bandwith, and the venet0:0 interface with an IP. If you run bare-metal or KVM or vserver etc. you can set these two to the same value (eth0 eth1 etc).

    # Network interface for the IP address
    iface="venet0:0"
    # network interface for traffic monitoring (RX/TX bytes)
    iface2="venet0"

The IP address of the server, this is used by me when deploying this script via chef or ansible. You can set it, but it is not required.

Services are checked by doing a *ps* to see if the process is running. The last service should be defined without a comma, for valid JSON. The code below monitors "sshd", "lighttpd", "munin-node" and "syslog". 

    SERVICE=lighttpd
    if ps ax | grep -v grep | grep $SERVICE > /dev/null; then echo -n "\"$SERVICE\" : \"running\","; else echo -n "\"$SERVICE\" : \"not running\","; fi
    SERVICE=sshd
    if ps ax | grep -v grep | grep $SERVICE > /dev/null; then echo -n "\"$SERVICE\" : \"running\","; else echo -n "\"$SERVICE\" : \"not running\","; fi
    SERVICE=syslog
    if ps ax | grep -v grep | grep $SERVICE > /dev/null; then echo -n "\"$SERVICE\" : \"running\","; else echo -n "\"$SERVICE\" : \"not running\","; fi
    #LAST SERVICE HAS TO BE WITHOUT , FOR VALID JSON!!!
    SERVICE=munin-node
    if ps ax | grep -v grep | grep $SERVICE > /dev/null; then echo -n "\"$SERVICE\" : \"running\""; else echo -n "\"$SERVICE\" : \"not running\""; fi

To add a service, copy the 2 lines and replace the SERVICE=processname with the actual process name:

    SERVICE=processname
    if ps ax | grep -v grep | grep $SERVICE > /dev/null; then echo -n "\"$SERVICE\" : \"running\","; else echo -n "\"$SERVICE\" : \"not running\","; fi

And, make sure the last service montiored does not echo a comma at the end, else the JSON is not valid and the php script fails.

Now setup a cronjob to execute the script on a set interval and save the JSON to the webserver directory.

As root, create the file /etc/cron.d/raymon-client with the following contents:

    SHELL=/bin/bash
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    */5 * * * * root /root/scripts/client.sh | sed ':a;N;$!ba;s/\n//g' > /var/www/stat.json

In my case, the client script is in */root/scripts*, and my webserver directory is */var/www*. Change this to your own setup. Also, you might want to change the time interval. \*/5 executes every 5 minutes. The sed line is in there to remove the newlines, this creates a shorter JSOn file, saves some KB's. The *root* after the cron time is special for a file in */etc/cron.d/*, it tells cron as which user it has to execute the crontab file.

When this is setup you should get a stat.json file in the /var/www/ folder containing the status json. If so, the client is setup correctly.

### Server 

The status server is a php script which fetches the json files from the clients every 5 minutes, saves them and shows them. It also saves the history, but that is defined [below](#History). 

Requirements:

- Webserver with PHP (min. 5.2) and write access to the folder the script is located.
- PHP option "allow_url_fopen" set to 1 (allow opening of remote files)

Steps:

Create a new folder on the webserver and make sure the webserver user (www-data) can write to it.  

Place the php file "stat.php" in that folder.  

Edit the host list in the php file to include your clients:  

The first parameter is the filename the json file is saved to, and the second is the URL where the json file is located.  

    $hostlist=array(
                    'example1.org.json' => 'http://example1.org/stat.json',
                    'example2.nl.json' => 'http://example2.nl/stat.json',
                    'special1.network.json' => 'http://special1.network.eu:8080/stat.json',
                    'special2.network.json' => 'https://special2.network.eu/stat.json'
                    );

Edit the values for the ping monitor:  

    $pinglist = array(
                      'github.com',
                      'google.nl',
                      'tweakers.net',
                      'jupiterbroadcasting.com',
                      'lowendtalk.com',
                      'lowendbox.com' 
                      );

Edit the threshold values:  

    ## Set this to "secure" the history saving. This key has to be given as a parameter to save the history.
    $historykey = "8A29691737D";
    #the below values set the threshold before a value gets shown in bold on the page.
    # Max updates available
    $maxupdates = "10";
    # Max users concurrently logged in
    $maxusers = "3";
    # Max load.
    $maxload = "2";
    # Max disk usage (in percent)
    $maxdisk = "75";
    # Max RAM usage (in percent)
    $maxram = "75";


#### History

To save the history you have to setup a cronjob to get the status page with a special "history key". You define this in the stat.php file:

    ## Set this to "secure" the history saving. This key has to be given as a parameter to save the history.
    $historykey = "8A29691737D";    

And then the cronjob to get it:
    
    ## This saves the history every 8 hours. 
    30 */8 * * * wget -qO /dev/null http://url-to-status.site/status/stat.php?action=save&key=8A29691737D

The cronjob can be on any server which can access the status page, but preferably on the host where the status page is located.

