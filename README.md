# Installation and initial configuration of the Counter-strike game server on a vServer

## Part 0. Foreword

### Copyright ©
This article was written by me personally using my personal experience and public sources that I provide links to at the end of each section. <br />
Distribution of this article is forbidden without reference to the original source and author's nickname (contact).<br />
The primary source is:<br /> [Nord1cWarr1or/cs-1.6-server-installation-on-vds-using-lgsm (github.com)](https://github.com/Nord1cWarr1or/cs-1.6-server-installation-on-vds-using-lgsm)<br />
Author: Nordic Warrior. (Telegram: @NordicWarrior)<br />
Date of writing: October-November 2021.

### A few words from the author
My motivation me to write this article was the lack of any detailed articles and instructions on setting up a server by the end of 2021.<br />
Everything I could find was either outdated, not detailed enough or focused on a single process in the server setup. so I decided to make a single guide with up-to-date information - fully detailed and understandable even for people unfamiliar with this field.

***Note***:<br /> It will only cover installation on Linux, and even more specifically on [Debian](https://www.debian.org/).<br /> I am used to working with it and the commands on the other major distributions differ slightly, so there is no point in stretching the article.<br /> The installation will be done by using the [LGSM](https://linuxgsm.com/) script, because it's the best free tool for Counter-strike 1.6 server management.

### Software we will need:
1. FTP client. I use [FileZilla](https://filezilla-project.org/)
2. SSH Client. I use [PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/)
3. (Optional) A good text editor. Peronally I prefer [Notepad++](https://notepad-plus-plus.org/)

## Part 1. Initial system setup
### Choosing a host...
...I leave it up to you. There are many hosting companies and they differ in price, equipment, location and so on.<br />
There is no point in discussing this aspect here. Some even use their own server at home...

The approximate characteristics of the Vserver should be:
- Processor: I recommend 2.5-3.5 GHz, 1 core per server. if you are going to run two servers, order two cores, etc.
- RAM: 2gb if you are going to install a http server and a MySQL database on two cores, otherwise 1gb will be enough.
- Hard disk (or SSD): 10gb is enough for 1-2 servers.

### Data and input
So we decided on a web hosting and rented a server. Some webhosts require you to manually start the installation of the system.<br />
To begin, go into the server control panel and look there for the data to connect to it. We are interested in the SSH connection and the IP address of the server.<br />
After we have the data (I recommend writing it down somewhere safe, such as a text document), open PuTTY and add the connection to our server there. Type of connection is SSH, enter the IP and password of your server.<br />
Connect!

Login with your SSH<br />This brings us to our account login page. We open the data we wrote down a minute ago and log in.

### Preparing the Vserver
Once we are logged into the root account, and before we start the initial setup of the vServer, we should update all packages with:
```
apt update && apt full-upgrade
```

and wait for the commands to complete.

***Note***: If your webhost does not have the latest OS distribution and you run, for example, Debian 11, then `apt update` may give you the following output:
```
E: Repository 'http://security.debian.org/debian-security buster/updates InRelease' changed its 'Suite' value from 'stable' to 'oldstable'
N: This must be accepted explicitly before updates for this repository can be applied. See apt-secure(8) manpage for details.
```
Here, the system will tell us that the repositories have changed their label from stable to oldstable and ask you to accept the change: 
`Do you want to accept these changes and continue updating from this repository? [y/N]`

Accept and press enter until the installation starts.

### Adding a user
Now we will go through the initial configuration of the Vserver.
First, we have to create a new user and give it administrator privileges, because it is not recommended to run as root for security reasons.
Type `adduser username`. For example:
```
adduser public_server
```
The system will ask us to enter, one by one, the password for the user, then confirm the password. Next, we are asked to enter the user information, but this is completely optional. Press enter one by one, waiting for the question `Is the information correct? [Y/n]`, then enter Y, enter.

Install package **sudo**, which is responsible for executing administrator commands not from a **root** user.
```
apt install sudo
```
Next, let's give the new user admin rights. 
Type ``usermod -aG sudo username``. We have it:
```
usermod -aG sudo public_server
```

### Configuring the firewall (UFW)
The next step is to do the installation and basic configuration of the firewall.
Those who want to use other network protection solutions, like iptables or hosting protection, can skip this step.

First, we will install the `UFW` package. Type in the terminal:
```
apt install ufw
```
This is our firewall. We will now proceed to configure it.
First, we will prohibit all incoming connections and allow outgoing ones.<br /> Execute the command:
```
ufw default deny incoming && ufw default allow outgoing
```
Now, we need to allow incoming connections, which we need for the server to work properly.
First of all, we should allow SSH, since this is actually access to the terminal on the server. Enter:
```
ufw allow ssh
````
Then allow http connections which we need for fast downloads (and web extensions).
```
ufw allow http
```
Those who are going to organize a full-fledged forum or a website, for example, will probably use an encrypted protocol, so they should allow https as well:
```
ufw allow https
```
And of course, we need to allow incoming connections on the port that the game server will use. In my case this is the standard - 27015. Execute:
```
ufw allow 27015
```
Now activate the firewall with the command:
```
ufw enable
```
The system will warn you that activation may interrupt the connection, but since we have configured everything correctly, we agree by pressing Y and enter.

This completes the firewall configuration. If you doubt its validity, you can verify it by running the `ufw status verbose` command. If the firewall is enabled, you will get a list of allow/deny rules.

Just in case, here are some basic commands to disable/reset the firewall.
- `ufw disable` - disabling.
- `ufw reset` - reset.
- `ufw status numbered` - displays a list of active rules and their numbers.
- `ufw delete rule number` - deletes a certain rule by its number.

#### Literature
- [Configuring a firewall with UFW in Debian 11 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-11-243261243130246d443771547031794d72784e6b36656d4a326e49732e)

## Part 2. Installing and configuring the LGSM script
Now on to the brain of our future server: the script that runs the server.

### Installing LGSM

#### Dependencies installation
Let's go into the user we created in the previous step: `su - public_server`.
Begin by installing the dependencies - packages that are required for the script to work properly. Enter into the terminal: 
```
sudo dpkg --add-architecture i386; sudo apt update; sudo apt install curl wget file tar bzip2 gzip unzip bsdmainutils python util-linux ca-certificates binutils bc jq tmux netcat lib32gcc1 lib32stdc++6
```
Enter the user password to confirm and wait for it to complete.

***Note***: On Debian 11, I received a warning that the `lib32gcc1` package in the repository has been replaced by the `lib32gcc-s1` package. If you get the same message, just replace the package name in the line above.

Next, run the command that loads the script into the current folder:
```
wget -O linuxgsm.sh https://linuxgsm.sh && chmod +x linuxgsm.sh && bash linuxgsm.sh csserver
```

##### Literature
- [Counter-Strike | LinuxGSM_](https://linuxgsm.com/lgsm/csserver/)

#### (Optional) Choose a name for the script
At this stage, if you want to set your own script name instead of the standard `csserver`, this can easily be done. Otherwise, you can skip this step.
I will not change the name of the script from the standard one below so that there is no confusion.

To rename the script, use the `mv` command:
```
mv csserver new_name
```
For example: ``mv csserver public``.

I will continue with the default `csserver` so that there is no confusion.

#### Installation process
Start the automatic installation with:
```
./csserver auto-install
```
The script will automatically check if all the dependent components are installed and if not, install them.
If the installation of the script and the server was successful, you will get the corresponding inscription about it.

Now you can try to run the server for testing.
Write in the terminal:
```
./csserver start
```
If everything is successful, you can try to connect to the server.
Use the IP of the Vserver and the standard port 27015.

***Note***: at this stage, you can only log in to the server from a licensed version of the game.
***For your information***: you should only **exit the console using the key combination `CTRL + B`, then `D`. If you try to exit the console by pressing the usual `CTRL + C`, this will end the server process.

#### Activate full crash logs
At this point, you can immediately activate the creation of full crash logs. Otherwise, if your server crashes, no one can help you. This is done quickly and easily.

Install package gdb through the terminal, write:
```
apt install gdb
```
Connect to the server via FileZilla, using the SFTP protocol. To do it, enter in the field "Host" the address of your server, username and password in the corresponding fields (don't forget, that we should log in now not as root, but as a user, who has our script installed), and in the field "Port" write "22", that corresponds to SFTP. Let's connect.
We get into the user's home directory, in the example I used `public_server`.

Then go to the folder`serverfiles` via FileZilla and find here the file `hlds_run`.
Open it for editing. Search for the following line:
```
ulimit -c 2000
```
And replace 2000 with "unlimited" to make it look like this:
```
ulimit -c unlimited
```
That's it! Save and close the file.


##### Literature
- [How to get HLDS Dump | Dev-CS.ru](https://dev-cs.ru/threads/1532/#post-18163)

### Configuring LGSM
Now we can move on to the settings of the script.
The first thing we need to do is configure the basic settings.

#### Server and script settings
Go through an FTP client to the folder `lgsm -> config-lgsm -> csserver`. This is where LGSM configs are stored.
We are interested in config `csserver.cfg`, which is responsible for our game server.
Open it and make the initial settings.

All available parameters can be seen in the standard config `_default.cfg`, which is in the same directory. The descriptions to them can be read at the links given in the config.

I will give you an example of my settings.
```
##################################
####### Instance Settings ########
##################################
# PLACE INSTANCE SETTINGS HERE
## These settings will apply to a specific instance.

#### Game Server Settings ####

## Predefined Parameters | https://docs.linuxgsm.com/configuration/start-parameters
ip="0.0.0.0"
port="27015"
clientport="27005"
defaultmap="de_dust2"
maxplayers="16"

## Server Parameters | https://docs.linuxgsm.com/configuration/start-parameters#additional-parameters
startparameters="-game cstrike -strictportbind +ip ${ip} -port ${port} +clientport ${clientport} +map ${defaultmap} +servercfgfile ${servercfg} -maxplayers ${maxplayers} -pingboost 3 -debug"

#### LinuxGSM Settings ####

## Backup | https://docs.linuxgsm.com/commands/backup
maxbackups="4"
maxbackupdays="30"
stoponbackup="on"

## Logging | https://docs.linuxgsm.com/features/logging
consolelogging="on"
logdays="7"

#### Directories ####
# Edit with care

## Game Server Directories
systemdir="${serverfiles}/cstrike"
executabledir="${serverfiles}"
executable="./hlds_run"
servercfgdir="${systemdir}"
servercfg="${servercfgdefault}"
servercfgdefault="server.cfg"
servercfgfullpath="${servercfgdir}/${servercfg}"
```
In general, in my opinion, everything is intuitively clear here, but still I think it is necessary to go through this configuration.
- The `ip` is responsible for the IP of the server. If at the previous step you were able to enter the server, leave it in its default form. `0.0.0.0` - stands for automatic detection of the IP address. Otherwise, you can try to put here directly the IP of the server.
- `port` - the desired port of the server. Do not forget to give access to it via UFW.
- `clientport` - the client port. Do not touch.
- `defaultmap` - The map from which the server will start.
- `maxplayers` - The number of slots on the server.
- `Startparameters` - Server startup parameters.
- `maxbackups` - maximum number of stored backups of the server.
- `maxbackupdays` - how many days each backup will be stored.
- `stoponbackup` - whether to stop the server during the backup. It is best to leave it on.
- `consolelogging` - log everything that happens in the console of the server. In my opinion, a very handy feature, which is not enough on the hosting.
- `logdays` - how many days the logs will be stored.
- `systemdir` - directory where the root folder of the game server.
- `executabledir` - directory where the root folder of the script itself is located.
- `executable` - name of the executable file of the script. I do not recommend to change the name itself, but you can add here startup parameters of the server, such as binding to a specific processor core.
- `servercfgdir` - directory where the game server config (server.cfg) is located.
- `servercfg` - name of the game server config. Here I changed the value to `` ${servercfgdefault}`` to use a more familiar name for CS 1.6 `server.cfg`. The standard here is `"${selfname}.cfg"` - the name of the config by the name of the script. The result was: `csserver.cfg`.
- `servercfgdefault` - standard name of config. Do not change it.
- `servercfgfullpath` - the full path to the config server.

After configuring the config, restart the server with the command `./csserver restart`, so that the settings are applied.

#### Script commands
It is best to learn right away what commands our script has at its disposal. To do this, type `./csserver` - then a list of all commands will appear with their descriptions.

I am going to translate here some of the main ones that we may need. 

Command | Abbreviation | Description
-|-|-
start | st | start server
stop | sp | stop server
restart | r restarts the server
monitor | m |- checks the availability of the server and restarts it if it crashes
test-alert | ta | sends a test warning (more on this later)
details | dt | shows basic information about the server
postdetails | pd | the same as the previous one, but it uploads the information to the Termbin service (analogous to Pastebin) and gives a link
update-lgsm | ul | checks and updates the LGSM script.
backup | b makes a backup of the server.
console | c Opens the console of the game server
send | sd | gives you the ability to send a command to the console of the server without entering it

#### Literature
- [Home - LinuxGSM_](https://docs.linuxgsm.com/)

### Configuring Schedules
Now it's time to do the most important part of our job: configuring the schedules to automate the actions really important for our server.

In a moment we are going to set up:
- Reboot the Vserver once a month.
- Automatic start of the server when the Vserver is switched on.
- Rebooting the game server once a day.
- Creating a backup copy of the server every two weeks.
- "Monitoring" the server for crashes.

Here we go.
We will use the standard Linux "crontab" utility.
First of all, let's login as root in the terminal: `su - root`. We will type in the password. Then we access the crontab edit window. Type:
```
crontab -e
```
If you are running this command for the first time on the current Vserver, the system will ask you to choose which text editor you prefer to use (and if you have more than one installed "out of the box" in your distribution). In my case, I was offered **Vim** or **nano**. For all newbies, I highly recommend choosing nano because Vim has very specific controls.

So, you have chosen the editor and a file containing cron jobs (that's what they are called) opens up in front of you. By default it is empty (apart from the "comments").
Note that each cron job is written on a new line.

Use the arrows on the keyboard to bring the input down to the very bottom.
Set the reboot of the Vserver. It makes more sense to do it in the morning, when the server is low online. I chose the time 4 am. 
```
0 4 1 * * reboot
```
Press CTRL + X, the system will offer to save our file with the tasks. Press Y.

We can check if we did it right, by typing the following command:
```
crontab -l
```
which will display the content of the file. If you see at the end the line you just typed, then everything is fine.

Next, to work with tasks on the game server, it is best to use the user under which our script runs. You can go to the account of this user and repeat the above steps, or you can simply edit the job file of that user without leaving root.

Let's write:
```
crontab -u username -e
```
This will open the same file, only later on it will be executed by our user running LGSM.

Let's set up an automatic start of the server after turning on the Vserver:
```
@reboot ./csserver st > /dev/null 2>&1
```
Set the game server to reboot once a day. Again, in the early morning hours:
```
0 4 * * * * ./csserver r > /dev/null 2>&1
```
Next, let's set a bi-weekly backup (at 04:05, after a reboot):
```
5 4 */14 * * * ./csserver b > /dev/null 2>&1
```
Finally, let's add monitoring the server for availability.
The `monitor` command, which will check the server for crashes, for example every 5 minutes:
```
*/5 * * * * * ./csserver m > /dev/null 2>&1
```
For reference: `> /dev/null 2>&1` is used to lock text output to the screen from running a command.

#### Literature
- [Home - LinuxGSM_](https://docs.linuxgsm.com/)
- Crontab.guru - The cron schedule expression editor (https://crontab.guru/) - good service for setting the time of a cron.

## Part 3. Installing the main add-ons. Game server setup
Now it's time to take care of the game server itself. So let's start with the engine.

### Installing ReHLDS.
Click on the link: [Releases - dreamstalker/rehlds (github.com)](https://github.com/dreamstalker/rehlds/releases). Download and install the latest release.

Installation is extremely simple: switch off our server with command
`./csserver sp` and just upload files via FileZilla to *the root server folder* (i.e. /serverfiles), replacing the current ones. Turn on the server, go to the console and check the version with the `version` command.

At the time of writing I get the following output:
```
Protocol version 48
Exe version 1.1.2.7/Stdio (cstrike)
ReHLDS version: 3.11.0.767-dev
Build date: 03:13:55 Oct 25 2021 (2753)
Build from: https://github.com/dreamstalker/rehlds/commit/471158b
```

#### Creating and configuring a config
Since ReHLDS does not have its own config file, we will create it with our own hands. Go to the cstrike folder and create the file `rehlds.cfg`. <br />Open it and fill it with these parameters [^1], which brings the engine exactly ReHLDS. And add a line, which will be displayed in the console when reading the config.
```
echo Executing ReHLDS Configuration File

// Configuration file for ReHLDS

// Automatically load sounds used in v_* models
sv_auto_precache_sounds_in_models "0".

// Load custom sprays after logging into the game, not when connected. This increases loading speed
sv_delayed_spray_upload "1"

//Put an attempt to use unknown commands into the console
sv_echo_unknown_cmd "1" 

// Allows to disable RCON password logging
sv_rcon_condebug "0"

// Fixes getting stuck on a moving platform/anti. (Global problem on DeathrunMod and on maps with escape vehicles)
sv_force_ent_intersection "0"

// Forcibly set client quar cl_dlmax to 1024. Helps to avoid excessive fragmentation of packets
sv_rehlds_force_dlmax "1"

// Sets the size of the entity in the center
sv_rehlds_hull_centering "0"

// Send mapcycle.txt in the serverinfo message (Not used on the client)
sv_rehlds_send_mapcycle "0"
       
// Fixes bug with player model animations when player has objects attached (aiments). May cause animation lag when cl_updaterate is low
sv_rehlds_attachedentities_playeranimationspeed_fix "0"

//limit the number of connections from one IP address
sv_rehlds_maxclients_from_single_ip "5"

// Allows you to use your own list of entities for cards. Entity file is located at "maps/[map name].ent")
// 0 - use the original enti.
// 1 - use .ent files from the maps directory.
// 2 - use .ent files from the maps directory and create a new .ent file if it is absent.
sv_use_entity_file "0"

// Local game time function, which reduces lag if you have the same map running for a long time
sv_rehlds_local_gametime "0"

// Maximum average "move" commands for the ban
sv_rehlds_movecmdrate_max_avg "400"
 
// Time in minutes for which the player will be banned (0 - forever, negative number - kick)
sv_rehlds_movecmdrate_avg_punish "-1"

// maximum deviation of the "move" commands for the ban
sv_rehlds_movecmdrate_max_burst "2500"

// Time in minutes for which the player will be banned (0 - forever, negative number - kick)
sv_rehlds_movecmdrate_burst_punish "-1"

// maximum average level of "string" commands for the ban
sv_rehlds_stringcmdrate_max_avg "80"

// Time in minutes for which the player will be banned (0 - forever, negative number - kick)
sv_rehlds_stringcmdrate_avg_punish "-1"

// maximum deviation of "string" commands for the ban
sv_rehlds_stringcmdrate_max_burst "400"

// Time in minutes for which the player will be banned (0 - forever, negative number - kick)
sv_rehlds_stringcmdrate_burst_punish "-1"

// setinfo fields to be passed to clients from the server.
// If not setinfo, all fields will be transmitted, except the prefix with an underscore (e.g. _ah). Each key must begin with a slash.
// For example "/name/model/*sid/*hltv/bottomcolor/topcolor"
// More information: https://github.com/dreamstalker/rehlds/wiki/Userinfo-keys
sv_rehlds_userinfo_transmitted_fields ""

// If enabled, the server will set an additional random number independently of the client. Used to break norecoil in cheats.
sv_usercmd_custom_random_seed "0"
```

Now open `server.cfg` and add the following lines at the bottom:
```
// Execute ReHLDS Config
exec "rehlds.cfg"
```
Finished!

***Note***:
If you are going to install ReGameDLL (the next step), it is better to add reading of ReHLDS config to its config - `game.cfg`. 
Also when updating ReGameDLL, if you update its config, don't forget about the lines with the ReHLDS config as well.

### Installing ReGameDLL
Next, let's install ReGameDLL./etc/nginx/sites-available
Repeat the previous steps, download the latest release from GitHub: [Releases - s1lentq/ReGameDLL_CS (github.com)](https://github.com/s1lentq/ReGameDLL_CS/releases).
Turn off the server, replace the files in the root of the server, turn it back on. Check your actions with the `game version` command in the console.
At the moment we will see the following:
```
ReGameDLL version: 5.21.0.540-dev
Build date: 17:33:16 Oct 25 2021
Build from: https://github.com/s1lentq/ReGameDLL_CS/commit/b9cccc6
```
Immediately, without leaving the box, you can configure for your server config `game.cfg `, which is responsible for gameplay on the server.
I will not disassemble it here, because, firstly, it would require a whole separate article, and secondly, it is too individual from server to server, and thirdly, it already has a signature for each quara.

Don't forget to add in `game.cfg` reading the ReHLDS config from the previous step.

### Installing Metamod
We are going to use Metamod-r because this version is a lot of optimizations, fixes and is fully compatible with ReHLDS.

Go to the link: [Releases - theAsmodai/metamod-r (github.com)](https://github.com/theAsmodai/metamod-r/releases) and download the latest available release.

Create a directory called addons underneath of cstrike and upload the metamod-folder.
Upload to the cstrike folder addons from the archive, and go to it, and then to the folder `/metamod`. Delete the file `metamod.dll`, because it is designed for Windows.
Here we also create a file `plugins.ini` - we will need it to install add-ons.

Now enable Metamod, to do this we return to the folder `cstrike` and look for the file `liblist.gam`, open it with your text editor.
We are interested in the parameter `gamedll_linux` - replace its value to the path to the Metamod .so-file:
`gamedll_linux "addons/metamod/metamod_i386.so"`

Reboot the server.

This completes the installation of Metamod.
Go to the console and make sure it works by entering the command `meta version`.
At the time of writing you should get this output:
```
Metamod-r v1.3.0.128, API (5:13)
Metamod-r build: 17:47:54 Aug 24 2018
Metamod-r from: https://github.com/theAsmodai/metamod-r/commit/0cf2f70
```

### Installing AmxModX
Here you have a choice: put the stable development branch (version 1.9.0) or experimental (1.10.0) where new functionality is introduced.
Personally I recommend 1.9.0, because if you are not famillary with AmxModX scripting, 1.10.0 version will not give you any benefits at all.

So, go to the link: [AMX Mod X - Half-Life Scripting for Pros!](https://www.amxmodx.org/downloads-new.php).<br />
Find the latest build, at the top of the table, click on the Linux icon, and download there **Base Package**, as well as **Counter-Strike**.
Open both archives, and download the addons folder from them into the cstrike folder, first from Base Package, then from Counter-Strike, agreeing to swap files.

Open the previously created by us file with a list of plugins for Metamod at the path: `addons/metamod/plugins.ini` and add there the following line:
```
linux addons/amxmodx/dlls/amxmodx_mm_i386.so
```
Save and restart the server.

Go into the console and make sure that AmxModX working by typing the command `meta list`. In the list should be an inscription like this:
```
[1] AMX Mod X RUN - amxmodx_mm_i386.so v1.9.0.5293 ini Start ANY
```
If it's there, it means we've done everything right and we can move on to the next step.

#### (Optional) Checking server crash dumps
Here you can also check if we have set up the display of server crash dumps correctly.

Create a new file in any location on our computer and name it, for example, `crash_test.sma`. Open it with a text editor and copy there the following code:
```
#include <amxmodx
#include <fakemeta>

public plugin_init()
{
    register_plugin("Crash", "1.0", "Dev-CS Team");

    //Generate exception code 0xC0000005
    set_task(1.0, "GenerateExceptionCode");
}

public GenerateExceptionCode()
{
    server_print("[Crash]: I call segmentation fault! Exception code: 0xC0000005");

    // Put invalid pointer that will generate access violation exception
    set_tr2(0xDEADBEEF, TR_InWater, true);
}
```
Save.

Next we have two options: compile the file on Windows or on Linux.
Those who know how to compile plugins on Windows can compile it and move on to the next step with installation.

For others, I will briefly explain how to compile a plugin on Linux.
Full article: [Local Compilation of Plugins | Dev-CS.ru](https://dev-cs.ru/threads/246/)

In FileZilla go to `serverfiles/cstrike/addons/amxmodx/scripting`.
Upload our `crash_test.sma` file here.
Find the file `amxxpc`, right-click on it and select *File permissions*. In the input field, type in the rights `754` -> OK.

Open PuTTY. Now we have to check that we are in the root folder of a user, with a command:
```
cd
```
Next, go to the folder where we put the source code of the crash test plugin:
```
cd serverfiles/cstrike/addons/amxmodx/scripting
```
Run the compilation:
```
./amxxpc crash_test.sma
```
If you see this output on the screen, then you've done everything right:
```
AMX Mod X Compiler 1.9.0.5293
Copyright (c) 1997-2006 ITB CompuPhase
Copyright (c) 2004-2013 AMX Mod X Team

Header size: 300 bytes
Code size: 288 bytes
Data size: 456 bytes
Stack/heap size: 16384 bytes
Total requirements: 17428 bytes
Done.
```
Do not forget to go back to the user's root folder with the `cd` command.

Next, expand FileZilla again, find in the same folder scripting our new file `crash_test.amxx` (not .sma!) and drag it to the two dots above to move it to the directory above.

Follow the standard plugin installation procedure: drag the file to the plugins folder, go to the configs folder, find the file `plugins.ini` there and paste at the very end: `crash_test.amxx`. Save it.

Now reboot our server and see what happens. In the terminal run: `./csserver r'.
***Don't forget to immediately disable the crash test plugin! Comment out or delete its name in `plugins.ini`.

Deploy FTP and go to the folder serverfiles. Here look for the file `debug.log` and open it.
If you see something like this content:
```

----------------------------------------------
CRASH: Sat 16 Oct 2021 07:27:08 AM MSK
Start Line: ./hlds_linux -game cstrike -strictportbind +ip 0.0.0.0 -port 27015 +clientport 27005 +map de_dust2 +servercfgfile server.cfg -maxplayers 16 -pingboost 3 -debug -pidfile hlds.5833.pid
[New LWP 5870]
[New LWP 5871]
[New LWP 5876]
[New LWP 5877]
[New LWP 5878]
[New LWP 5879]
[New LWP 5880]
[New LWP 5881]
[New LWP 5872]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Core was generated by `./hlds_linux -game cstrike -strictportbind +ip 0.0.0.0 -port 27015 +clientport'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0xf265935b in set_tr2(tagAMX*, int*) () from cstrike/addons/amxmodx/modules/fakemeta_amxx_i386.so
[Current thread is 1 (Thread 0xf7b8c700 (LWP 5870))]
#0  0xf265935b in set_tr2(tagAMX*, int*) () from cstrike/addons/amxmodx/modules/fakemeta_amxx_i386.so
#1  0xf293bb08 in CLog::LogError(char const*, ...)::msg () from /home/cs/serverfiles/cstrike/addons/amxmodx/dlls/amxmodx_mm_i386.so
#2  0xf24f9454 in ?? ()
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
No symbol table info available.
From        To          Syms Read   Shared Object Library
0xf7f5b130  0xf7f5c1c4  Yes (*)     /lib/i386-linux-gnu/libdl.so.2
0xf7ecd914  0xf7f13c78  Yes         ./libstdc++.so.6
0xf7d8b170  0xf7e4c4af  Yes (*)     /lib/i386-linux-gnu/libm.so.6
0xf7bbc0e0  0xf7d08d76  Yes (*)     /lib/i386-linux-gnu/libc.so.6
0xf7f6f090  0xf7f8a50b  Yes (*)     /lib/ld-linux.so.2
0xf7b8fe04  0xf7b9f490  Yes         ./libgcc_s.so.1
0xf74b02b0  0xf75a0660  Yes (*)     /home/cs/serverfiles/engine_i486.so
0xf74733d0  0xf7476cb4  Yes (*)     /lib/i386-linux-gnu/librt.so.1
0xf74605c0  0xf746ad74  Yes (*)     ./libsteam_api.so
0xf74425e0  0xf7451eff  Yes (*)     /lib/i386-linux-gnu/libpthread.so.0
0xf7417e00  0xf7433838  Yes (*)     /home/cs/serverfiles/filesystem_stdio.so
0xf567e000  0xf6bdb0c4  Yes (*)     /home/cs/.steam/sdk32/steamclient.so
0xf2cfa6a0  0xf2d3cf70  Yes (*)     /home/cs/serverfiles/./cstrike/addons/metamod/metamod_i386.so
0xf2a95280  0xf2c2c0b0  Yes (*)     /home/cs/serverfiles/cstrike/dlls/cs.so
0xf2779a20  0xf27dcdb7  Yes (*)     /home/cs/serverfiles/cstrike/addons/amxmodx/dlls/amxmodx_mm_i386.so
0xf2702b60  0xf274a624  Yes (*)     cstrike/addons/amxmodx/modules/hamsandwich_amxx_i386.so
0xf2671120  0xf2677404  Yes (*)     cstrike/addons/amxmodx/modules/csx_amxx_i386.so
0xf262f9d0  0xf265bd6c  Yes (*)     cstrike/addons/amxmodx/modules/fakemeta_amxx_i386.so

0xf0328000  0xf18468a4  Yes (*)     ./steamclient.so
0xf2260670  0xf22d6020  Yes (*)     ./crashhandler.so
0xf24db300  0xf24e1cd4  Yes (*)     /lib/i386-linux-gnu/libnss_files.so.2
0xf24d21c0  0xf24d51f4  Yes (*)     /lib/i386-linux-gnu/libnss_dns.so.2
0xf24ba3a0  0xf24c6014  Yes (*)     /lib/i386-linux-gnu/libresolv.so.2
(*): Shared library is missing debugging information.
Stack level 0, frame at 0xff9c4734:
 eip = 0xf265935b in set_tr2(tagAMX*, int*); saved eip = 0xf293bb08
 called by frame at 0xff9c4738
 Arglist at 0xff9c472c, args: 
 Locals at 0xff9c472c, Previous frame's sp is 0xff9c4734
 Saved registers:
  ebx at 0xff9c4728, ebp at 0xff9c472c, esi at 0xff9c4720, edi at 0xff9c4724, eip at 0xff9c4730
End of crash report
----------------------------------------------
```
Then you have succeeded.
Otherwise you'll see a truncated crashlog:
```
----------------------------------------------
CRASH: Wed 13 Oct 2021 09:16:04 PM MSK
Start Line: ./hlds_linux -game cstrike -strictportbind +ip 0.0.0.0 -port 27015 +clientport 27005 +map de_dust2 +servercfgfile server.cfg -maxplayers 16 -pingboost 3 -debug -pidfile hlds.20701.pid
End of crash report
----------------------------------------------
```
Try re-reading and re-executing the step with full crash logs activated.

##### Literature
- [How to get Dump of HLDS drop | Dev-CS.ru](https://dev-cs.ru/threads/1532/)
- [Local compilation of plugins | Dev-CS.ru](https://dev-cs.ru/threads/246/)

### Configuring fast downloads
So, to make our server work properly we need to set up fast file uploads (FastDL).
Otherwise, unless you have a standard server with extra resources, nobody wants to download them for 10 minutes.

Let's start by installing the Nginx web server. Let's write:
```
sudo apt install nginx
```
After installation it will automatically start. You can check this by going to our server address in your browser. If you see a page with the text: `# Welcome to nginx! 

Now move on to configuring Nginx for the fast server.
To do this, log in as root user via FileZilla, and go to the following path:<br /> `/etc/nginx/sites-available`. Here will be the default nginx config file (`default`).
Open it for editing. In the section **server**, before closing bracket '}' insert the following:
```
	# Fast load for Counter-Strike
	location /cstrike/ {
		alias /home/public_server/serverfiles/cstrike/;
		autoindex on;

		location ~* (\.wad$||(maps|sprites|models|gfx|sound|media|overviews)/.*(bsp|mdl|spr|wav||mp3|bmp|tga|txt||res)$) {
			allow all;
		}

		deny all;
	}
```
Where `/home/public_server/serverfiles/cstrike/` is the path to the cstrike folder on our server. Check it and correct it to your version if it's different.
Save this file and make sure it is correct. To do this, we will need this command:
```
sudo nginx -t
```
If we get a message that all configs are OK, restart Nginx:
```
`` sudo service nginx restart
```
Now, we can check what we have done. Open the browser again, go to our web server and try to download some file, for example the map **de_dust2**.
```
http://SERVER_ADDRESS/fastdl/maps/de_dust2.bsp
```
If it downloaded, then you did everything right!

Now open the game server config (**server.cfg**) and add the following cvars there:
```
sv_allowdownload 1
sv_downloadurl "http://SERVER_ADDRESS/fastdl/"
```

This completes the basic configuration of the server!

#### Literature
- [How to install LAMP on Debian 11](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mariadb-php-lamp-stack-on-debian-11)
- [Configuring FastDL correctly on Vserver (c-s.net.ua)](https://c-s.net.ua/forum/topic67228s60.html#entry1045699)

## Part 4. Additional tools for the server. Recommendations

### Installing and configuring Reunion
It is most likely that you will be visited by players with a pirated copy of the game (Non-steam). It may even be you yourself. However, they will not be able to connect to the server without a special module, which provides the game on the server for pirate clients.

You can download Reunion from here:
- [CS.RIN.RU - Steam Underground Community - Topic Overview - Reunion 0.1.92 - no-steam for ReHLDS](https://cs.rin.ru/forum/viewtopic.php?f=29&t=69235)
- [addons Reunion | Dev-CS.ru](https://dev-cs.ru/resources/68/)

Requires registration, but it's a reliable source, which I can vouch for.
At the time of writing this article the stable version is 0.1.92d.

Place binary file `reunion_mm_i386.so` on path `addons/reunion/`, then write in Metamod-plugins file:
```
linux addons/reunion/reunion_mm_i386.so
```
After that put the configuration file ``reunion.cfg`` from the archive into the folder cstrike.

**Important**!<br />
Reunion must be at the first place in the list of meta plugins.

**Important!**:<br />
Even if you don't intend to do a deep setup of Reunion, it is highly recommended to set "salt". This is necessary so that no one can take the SteamID of another player.
Find the line `SteamIdHashSalt` in the config and enter there a set of random characters with a length of at least 16 characters.

Restart the server for the module to start its work.
Now we will make sure that it works. In the server console, we will write the familiar command `meta list`. Reunion must have a status of **RUN**:
```
Currently loaded plugins:  
description stat pend file vers src load unlod  
[ 1 ] Reunion RUN - reunion_mm_i386.so vX.X.X ini Start Never
```

This completes the installation. 

**"Important!"**:<br />
For more fine-tuning Reunion, there's an excellent guide at this link: [SteamID Switching | Dev-CS.ru](https://dev-cs.ru/threads/808/) <br />
If you're going to maintain a dedicated server for everyone, and not just play with friends or bots, I recommend doing the setup from the article above.

### Installing web server components
Most server owners will probably want to install CSBans or other web add-ons. The basic tools you need to accomplish this are: PHP and MySQL.

#### Installing MariaDB
We will start by installing MariaDB[^2].

We enter the command:
```
sudo apt install mariadb-server
```
After installing, we need to do the configuration with the command:
```
sudo mysql_secure_installation
```
Now we will be asked to specify the password for the root user MariaDB, if it was already specified. Since we have just installed this DBMS, leave the field blank and hit enter.

In some cases at this point, the system may offer to `switch to unix_socket authentication [Y/n]`
. Override.
Now we will be asked to change the root password (`Change the root password? [Y/n]`). We will agree and set our own password.

Otherwise, if you are prompted for a root password (`Set root password? [Y/n]`), accept and set it.

Next is the standard setup procedure, the system will ask you to remove anonymous users, disable remote root access, delete the test database and reload the privilege table. In all variants you can choose **Y**.

This completes the installation.

But now we need to create a new MySQL user to use it from under the game server.
Log in to our database as a root user:
```
mysql -u root -p
```
Now create a user (admin - login and password - specify your own):
```
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'password';
```
Then give this user all privileges:
```
GRANT ALL PRIVILEGES ON * .* TO 'admin'@'localhost';
```
And write `exit` to exit the database.

This completes the installation of the database!
##### Literature
- [How to Install MariaDB Database in Debian 11 (digitalocean.com)](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-debian-11)

#### Installing PHP
This will barely take you any time at all because PHP does not need any extra configuration.

Type it into the console:
```
apt install php-fpm php-mysql
```
This completes the installation. But just in case, check everything is in order.
Go to FileZilla under the following path: `/var/www/html`.
Here we create the file `info.php`. Open it and insert the following content:
```
<?php

 phpinfo();

?>
```
Save.

In the browser we go to the address of our file, ie: `http://SERVER_ADDRESS/info.php`.
If you see the page with the php configuration, then you have done everything correctly.

##### Literature
- [How to install LAMP on Debian 11](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mariadb-php-lamp-stack-on-debian-11)

#### (Optional) Installing Phpmyadmin
If you don't need the MySQL database control panel, you can skip this step.

First, let's install the necessary components:
```
apt install php-mbstring mcrypt
```
Now proceed to install the utility itself:
```
apt install phpmyadmin
```
At first we will be asked to choose a server to work with the application. Unfortunately Nginx is not on the list, so deselect both and click OK.
The next step will automatically create databases for phpmyadmin, we agree.
After that, we will be asked to enter a password for the user phpmyadmin. Enter it.

Then we need to enable the extension `mcrypt`, which we do:
```
phpenmod mcrypt
```
And the last step. Open the familiar Nginx config by going to `/etc/nginx/sites-available`.
Find the following line:
```
	# Add index.php to the list if you are using PHP
	index index.html index.htm index.nginx-debian.html;
```
Add `index.php` to that list. It should look like this:
```
	# Add index.php to the list if you are using PHP
	index.php index.html index.htm index.nginx-debian.html;
```
Then we put the phpmyadmin config at the end of the shared section:
```
	# phpmyadmin config
	location /phpmyadmin {
		alias /usr/share/phpmyadmin/;

		location ~ \.php$ {
			fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
			fastcgi_index index.php;
			fastcgi_param SCRIPT_FILENAME $request_filename;
			include fastcgi_params;
			fastcgi_ignore_client_abort off;
		}

		location ~* \.(js|css|png|jpg|jpeg||gif|ico)$ {
			access_log off;
			log_not_found off;
			expires 1M;
		}
	}
```
Save the file. As usual, check the configuration with the `nginx -t` command, fixing errors if any. Reboot Nginx.
That's all!

Now let's go see what we have. In your browser we go to the link: `http://SERVER_ADDRESS/phpmyadmin`. If you see a login page to phpmyadmin then the installation was successful.

##### Literature
- [How to install phpMyAdmin on Debian 11](https://www.howtoforge.com/how-to-install-and-secure-phpmyadmin-on-debian-11/)

#### (Optional) Setting up automatic database backups
To avoid unforeseen situations, you can also set up a MySQL database backup.

We will use the tool `mysqldump` and already familiar to us `crontab`.

In my opinion, it is more convenient to store backups in the directory server, so we have to go to the user who has the server. In our case this is `public_server`.

Open the crontab editor and write the following line at the very end of the file:
```
0 12 */7 * * * mysqldump -u admin -ppassword -A > db.sql
```
Syntax:
```
mysqldump -u [mysql user] -p[his password] -A > [database name].sql
```
Note that `-p` followed by the user's password is written together.

How to configure the time in crontab was described in the "Configuring Schedules" section.

In the example we have the following:<br /> *Every seventh day of the month (once a week) at 12 noon a dump will be created for mysql user admin with password password and placed in the root folder of the user for whom crontab is running (in our case it is `public_server`), named db.sql*.

You can place the file in a directory, but in this case it must be **pre-created**. Example command:
```
mysqldump -u admin -ppassword -A > mysql_backup/db.sql
```

#### (Optional 2) Setting up automatic database backups for all databases
Install the automysqlbackup-package from Debian
```
sudo apt install automysqlbackup
```
These Backups will be automatically created daily in `/var/lib/automysqlbackup/`

##### Literature
- [How to back up a database/table in MySQL](https://devanswers.co/backup-mysql-databases-linux-command-line/)
- [How to backup automatically with automysqlbackup](https://serverpilot.io/docs/how-to-back-up-mysql-databases-with-automysqlbackup/)

### Enabling alerts on server malfunction
In LGSM it is also possible to turn on alerts if the `monitor` command detected server inaccessibility.

There is quite a long list of messengers and services for alerts, but they are more popular in the West than in the CIS countries.
For us, we can highlight Email, Telegram and Discord. The full list is available here: [Alerts - LinuxGSM_](https://docs.linuxgsm.com/alerts).

Here I will look at the connection to Telegram.

#### Alerts via Telegram
So, the first thing we need to do is to create a bot in Telegram.

Follow this link [Telegram: Contact @BotFather](https://t.me/BotFather) or search for @BotFather on Telegram if you don't have it installed on your computer.

Launch this bot and run the `/newbot` command.
Following the instructions, give the bot a name and nickname. As a result, we will have a bot token, which we will need to specify in the script settings.

Open the LGSM config by the familiar path: `lgsm/config-lgsm/csserver/csserver.cfg`.
At the very end we add:
```
## Notification Alerts
# (on|off)

# More info | https://docs.linuxgsm.com/alerts#more-info
postalert="on"

# Telegram Alerts | https://docs.linuxgsm.com/alerts/telegram
# You can add a custom cURL string eg proxy (useful in Russia) in "curlcustomstring".
# For example "--socks5 ipaddr:port" for socks5 proxy see more in "curl --help".
telegramalert="on"
telegramtoken="token"
telegramchatid="chatid"
```
So let's look at the parameters listed here.
- ``postalert`` - allows to get additional information about the server at the time of failure and uploads it to Termbin, similarly to the ``./csserver postdetails`` command.
- `telegramalert` - actually enable or disable alerts in Telegram.
- `telegramtoken` - insert the token of our new bot here.
- `telegramchatid` - here you should specify the index of your chat with the bot. About this below.

Insert the bot index obtained in the previous step in the `telegramtoken` parameter.
Then in the browser is the following path:
```
https://api.telegram.org/botXXXXXXXXX:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/getUpdates
```
Where `XXXXXXXXXXX:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX - replace it with the bot token. For example:
```
https://api.telegram.org/bot3104437352:AAE2XjHxOIBbfM2U1HStmcOkLdCTw5DJsjY/getUpdates
```
Leave tab open.

Now write any message to the bot, for example ``LGSM test``. Then refresh the page and we should see the technical parameters of the message.
Look for our message at the end of the line and go to the beginning to find data of this kind: `"chat":{"id":845483018,`.
The number `845483018` - this will be the index of the chat we need. Copy it and paste it into the `telegramchatid` parameter.

Close the config and save it. This configuration of notifications is complete. 

We can test their performance. To do this, use the script command `./csserver test-alert`.
As a result of its execution our new bot with the test alert should write to us in Telegram.

##### Literature
- [Telegram - LinuxGSM_](https://docs.linuxgsm.com/alerts/telegram) <br />
***Note***: Here you will also find instructions on how to connect the bot to a group instead of just a private chat.

***Note***: At the time of writing this article, the telegram alert is giving you an error.
To fix it I used the code from the previous commit in the necessary file (`alert_telegram.sh`).
I also sent a bug report to the developers. Hopefully they will fix it soon.

# That will be all. Good luck creating a server.
[^1]: CVar - Console Variable
[^2]: DBMS - Database Management System
