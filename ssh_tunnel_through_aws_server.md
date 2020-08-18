# ssh Tunneling For Remote Access Over The Public Internet Through an AWS Server

## Introduction

This document covers how to use a free-tier AWS EC2 server to configure remote tunneling for the purpose of remote access to a clinet computer for an administrative computer. This will allow an encryprted connection without having to configure any port forwaring on either of the local networks involved. Both computers are running Ubuntu 20.04, and the server is running Ubuntu as well. 

To avoid the need (and security risks) of opening a port to either of the local area networks, both the remote box (referred to as remote box or just box) and the admin machine (referred to as admin), will independently log into the intermediary server. The remote box will then have to be scheduled to run a script to do so, as we will not always have direct access (thus this configuration). The admin machine could just as well do this manually though, so this function, as well as the final tunnel to the remote box, will be coded into an elective script on the admin machine. The diagram below will outline the structure of the tunnel that will be created.

![tunnel diagram](ssh_tunnel_through_aws_server.images/tunnel_diagram.pdf)

Three sets of ssh keys will need to be created to create the tunnel illustrated above. The first has a private key (we'll call it `rsa_admin-server`) on the admin machine, and the corresponding public key in `~/.ssh/authorized_hosts` on the AWS server. The next would be created for the remote box, created on that machine with the private key (`rsa_box-server`) stored in `~/.ssh/` on the box, with the corresponding public key also on `authorized_hosts` on the server. The third is for the complete tunnel between the admin machine and remote box. It's generated on the admin machine, with the private key (`rsa_admin-box`) stored in `~/.ssh` on that machine and `~/.ssh/authorized_hosts` on the remote box. 

Line A in the above diagram illustrates the ssh connection between the remote box and the server. The key that allows this connection is `rsa_box-server`. It will be this connections will be scripted to the repeated periodically whenever the box is turned on, in case of disconnection.

Line C illustrates the ssh connection between the admin machine and the server. The key that allows this connection is `rsa_admin-server`.

Finally Line C illustrates the final ssh connection from the admin machine to the remote box, through the server. It continues from port 2222 on the admin machine to port 2223 on the serverr, to port 2223 on the remote box, to port 22 (and thus the ssh server) on the remote box. The key that allows this connection is `rsa_admin-box`.

## Set Up A Free-tier AWS EC2 Server Running Ubuntu

See [documentation on creating a free instance of an Ubuntu server on AWS](./set_up_free_aws_ubuntu.md).

## Configuring All The ssh Keys

There are three sets of keys that must be created as described in [the introduction](#introduction). 

Depending on how the AWS server was set up, keys to access it from the machine on which it was created (let's assume it's the admin machine) will be generated at the time of its creation and offered for download once only, as the means of accessing the instance. If the server already existed, then just set them up the same as the keys subsequently described.

To create the `rsa_box-server` keys (line A) navigate to `~/.ssh` on the remote box, and run `ssh-keygen`. Name the keys `rsa_box-server` with an empty password. Copy the text of the public key created by this process (`~/.ssh/rsa_box-server.pub`) into the file `~/.ssh/authorized_keys` on the server, connecting to it in whatever manner has been configured. If this file does not exist, create a plain text file of this name and paste the text of the `rsa_box-server` into it.

Repeat the same steps on the admin machine, naming the keypair `rsa_admin-server`. Copt the text in `rsa_admin-server.pub` into `~/.ssh/authorized_keys` on the remote box.

## Manually Setting Up A Cascading ssh Tunnel From Admin Through Server To Box

You should now have all the keys set up to be able to execute the following command to set up a connection from the admin machine to the server:

`ssh -L 2222:127.0.0.1:2223 -i ~/.ssh/rsa_admin-server ubuntu@123.123.123.123`

The L flag will configure local forwarding (the address 127.0.0.1 of course signifies localhost, or the computer the user is currently logged into) from local port 2222 to port 2223 on 123.123.123.123 (the aws server.

Run the following command on the remote box:

`ssh -R 2223:127.0.0.1:22 -i ~/.ssh/rsa_box-server ubuntu@123.123.123.123`

The R flag will configure remote forwarding from the server's port 2223 to the box's port 22. Thus, traffic routed to port 2223 on the server (tunneled from port 2222 on the admin machine as per the previous command) will go to the box's defualt ssh port 22.

This allows us finally to run the following command on the admin machine to complete the full tunnel from the admin machine to the remote box, bringing up a prompt for the remote box on the admin machine:

`ssh -p 2222 -i ~/.ssh/rsa_admin-box user@127.0.0.1`

This will open an ssh session with traffic routed to the local machine's port 2222 (which will then be tunneled to the server's port 2223 as per the first command and from there to the box's port 22 as per the second command). The user for this session should be specified as the name of the user account on the remote box that has been configured to have keys to the server, whichever that may be. It's reccomended but not necessary to use a non-privileged account for this purpose, and a more privileged account can then be accessed from this session without a problem. If there are any problems with the execution of this command, up to 3 -v verbosity tags can be added to increase verbosity, as the following command:

`ssh -vvv -p 2222 -i ~/.ssh/rsa_admin-box user@127.0.0.1`

## Automating The Establishment of The Tunnel

The below cited Robert Elder tutorial included a script to schedule on the remote box in order to automatically establish the tunnel, but it didn't work well for me. It took some tinkering to found something that did. To increase resilience, I decided to use a utility called `autossh` instead of the stock ssh to script the connection from the remote box to the server, and it seems to be a bit more resilient. To install `autossh` on an Ubuntu-based machine that doesn't have it, run the following command from an account with administrative privileges and follow the prompts:

`sudo apt-get install autossh`

Essentially the above command is scripted and put into a cron job to be executed periodically, to make sure that the connection is always established or reestablished from the remote box to the server, as long as it's on and the user there is logged in. So we take the following command:

`ssh -R 2223:127.0.0.1:22 -i ~/.ssh/rsa_box-server ubuntu@123.123.123.123`

We'll use `autossh` instead and also add a -N tag. More information on this tag can be found in ssh's man page, it basically runs the command without returning a prompt and is often used for setting up tunnels non-interactively in the background. I encoded this in the following script:

```
{
#! /bin/bash
LOCAL_SSH_PORT=22
REMOTE_SSH_PORT=2223
LOCAL_SSH_ADDRESS=127.0.0.1
REMOTE_SERVER=123.123.123.123
ID_FILE_BOX_SERVER=~/.ssh/rsa_box-server

autossh -N -R ${REMOTE_SSH_PORT}:${LOCAL_SSH_ADDRESS}:${LOCAL_SSH_PORT} -i ${ID_FILE_BOX_SERVER} ubuntu@${REMOTE_SERVER}
}
```

This script can be automated to run using cron. Assume it's saved as `.setup_ssh_admin_tunnel` on the remote box. A crontab can be added by running the following command on the remote box and placing a string encoding the desired frequency at the beginning of a new line at the end of the resulting text file, followed by the desired command:

`crontab -e`

This should pull up a text editor with the crontabs file. If it is the first time writing crontabs on the machine, it may prompt you to select the text editor of your choice. Add the following line at the bottom of the file.

`*/10 * * * * bash .setup_ssh_admin_tunnel`

Save and exit the editor, and the terminal should return the prompt and confirm that the crontab has been added. Wait about ten minutes.

A cheat sheet for the frequency encodings can be found [here](https://devhints.io/cron) or by searching 'crontab cheat sheet' in a search engine. I tried a few values for the frequency of this execution (@reboot seemed the most intuitive but didn't work for some reason). For troubleshooting at times I would run it every minute (`* * * * *`). I've been using a frequency of every ten minutes (`*/10 * * * *`), it provides a reasonable balance between availability and sparing use of system resources. The Robert Elder tutorial included conditional logic to determine if the tunnel is already running and foregoing the command if so, which seems ideal. However, subsequent running of the command doesn't seem to interfere with the established tunnel, and has been working well.

Establishing the tunnel on the admin machine could be configured as a crontab in the same manner, or run manually, with the following script. Assume it's saved as `~/.ssh/establish_remote_box_tunnel`:


```
{
#! /bin/bash

LOCAL_SSH_PORT=2222
REMOTE_SSH_PORT=2223
LOCAL_SSH_ADDRESS=127.0.0.1
REMOTE_SERVER=123.123.123.123
ID_FILE_ADMIN_SERVER=~/.ssh/rsa_admin-server
ID_FILE_ADMIN_BOX=~/.ssh/rsa_admin-box

ssh -N -L ${LOCAL_SSH_PORT}:${LOCAL_SSH_ADDRESS}:${REMOTE_SSH_PORT} -i ${ID_FILE_ADMIN_SERVER} ubuntu@${REMOTE_SERVER}
}
```

This can then be run either periodically from a crontab entry through the same method as above, or run manually with the command `bash ~/.ssh/establish_remote_box_tunnel`

After about ten minutes running the following command on the admin machine should establish a tunnel from the admin machine to the remote box:

`ssh -p 2222 -i ~/.ssh/rsa_admin-box user@127.0.0.1`

This can be encoded into the following script (`~/.connect_remote_box`):

```
{
#! /bin/bash

LOCAL_SSH_PORT=2222
REMOTE_SSH_PORT=2223
LOCAL_SSH_ADDRESS=127.0.0.1
REMOTE_SERVER=123.123.123.123
ID_FILE_ADMIN_SERVER=~/.ssh/rsa_admin-server
ID_FILE_ADMIN_BOX=~/.ssh/rsa_admin-box

ssh -p ${LOCAL_SSH_PORT} -i ${ID_FILE_ADMIN_BOX} mom@${LOCAL_SSH_ADDRESS}
}
```

This can be executed with `bash ~/.connect_remote_box`, and a prompt for the user on the remote box should return.

## Troubleshooting Steps In Case

Running `tail -f -n 500 /var/log/auth.log` on the server will display a log of authentication attempts, updating as they come in. This will yield server-side information on what's going on with the tunnel as you try to connect, as well as probably documenting unauthorized attempts at access to the server by strangers every 5-10 mintues from startup forever.



references:

https://blog.robertelder.org/amazon-cloud-servers-for-beginners-console-command-line/

http://www.augustcouncil.com/~tgibson/tutorial/tunneling_tutorial.html

https://tecadmin.net/crontab-in-linux-with-20-examples-of-cron-schedule/

https://medium.com/maverislabs/proxyjump-the-ssh-option-you-probably-never-heard-of-2d7e41d43464

https://www.ssh.com/ssh/tunneling/

https://blog.robertelder.org/amazon-cloud-servers-for-beginners-console-command-line/
