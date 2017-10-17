## Setup AWS account
Go to [https://aws.amazon.com/free/](https://aws.amazon.com/free/) to sign up for a free AWS account.

![AWS free tier page](https://serrintine.github.io/StreamDoc/img/awsfreetier.png "AWS free tier page")

You will need to complete registration by filling in an extra form, which should be sent to your email. You will need to enter credit card info and receive a phone call for verification purposes, but you should not be charged any payment.

## Create an EC2 instance
Once your account is running you should be able to see a dashboard of all the services AWS provides. Scroll down to **Build a solution** and click on "Launch a virtual machine" to start the steps for creating a free EC2 instance.
![Build a solution](https://serrintine.github.io/StreamDoc/img/buildasolution.png "Build a solution")

Choose **Ubuntu** as the server OS. You will by default be alloted the free tier server configuration, which is a **t2.micro** instance with 1GB memory and 8GB storage.

Once your instance is created you should be able to view the dashboard:
![Dashboard](https://serrintine.github.io/StreamDoc/img/ec2dashboard.png "Dashboard")

## Edit security group
Before you can actually access your server from your home computer, you need to create some firewall rules in your server's security group.

In the left hand side menu, navigate to **NETWORK & SECURITY** and click on the **Security Groups** item. Your server's privacy group should be automatically selected. Right click on it and select "Edit inbound rules".
![Security Groups](https://serrintine.github.io/StreamDoc/img/securitygroups.png "Security Groups")

You'll want to add the following rules:
![Inbound Rules](https://serrintine.github.io/StreamDoc/img/inboundrules.png "Inbound Rules")

Port 22 is the standard SSH port. Port 1935 is for the RTMP server. Port 5001 is for iperf to assess your server's network bandwidth (this is extra and totally not necessary for just setting up a stream so feel free to not include).

## Install Bash on Ubuntu on Windows 10 (or Putty)
If you have Windows 10, follow the [instructions here](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) to get a Bash shell working on your system. Everything after this section involves issuing text commands in a terminal and Bash is by far a superior command line experience than the default Windows command prompt.

Make sure that you pick **Ubuntu** as your Linux distribution of choice.

If you do not have Windows 10 or do not wish to install Bash on Ubuntu on Windows 10, then follow the [instructions here](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html) to use Putty to access your shiny new server.

The Putty method is very annoying and long-winded. I suggest just installing the Bash prompt.

## Setup RTMP server
First log into the server you've created. Linux servers are managed via SSH, which stands for secure shell. You get to issue text commands, and everything that happens between you and the server is encrypted. 
```bash
chmod 400 <key file>
ssh -i <key file> <user>@<server>
```
For example:
![SSH](https://serrintine.github.io/StreamDoc/img/ssh.png "SSH")

First get an updated list of software you can get through the package manager. Sudo runs a command as root (all the priviledges), and we need those privileges. You'll need Mercurial (to get the Nginx sources) and Git (for the RTMP module sources). The other things in the list are software packages that Nginx needs.

```
sudo apt-get update
sudo apt-get -y install mercurial git build-essential libpcre3-dev zlib1g-dev libssl-dev
```

Now that the prerequisites are in place, you'll need to grab the Nginx and RTMP module sources. Nginx is a web server program, and the RTMP module's an extension to Nginx that tells it how to handle video streams. 

Nginx developers store and track their code with Mercurial. To grab the code:
```
hg clone http://hg.nginx.org/nginx/
```

The RTMP module is hosted in Github. To grab the source code for that:
```
git clone https://github.com/arut/nginx-rtmp-module
```

Change into the Nginx source directory, and run a script that sets up the build process with the RTMP module:
```
cd nginx
auto/configure --add-module=../nginx-rtmp-module
```
That should output a lot of things, and complete without errors:
![configure output](https://serrintine.github.io/StreamDoc/img/postconfig.png "configure output")

Then compile Nginx with the RTMP module:
```
make
```
It's going to run for a bit, and finish like this:
![compile output](https://serrintine.github.io/StreamDoc/img/compiledone.png "compile output")

Get everything copied to the right place:
```
sudo make install
```

## Edit config file
Nginx needs to be told what to do with a config file. That's a set of directions on how it should operate. First get to where the config file is, and open it with nano.

Nano's like notepad hard mode. You can't use your mouse.
```
cd /usr/local/nginx/conf
sudo nano nginx.conf
```
You should see something like this:
![nano](https://serrintine.github.io/StreamDoc/img/nginxconfwithnano.png "nano")

Use arrow keys to get to the bottom of the file,
![nano](https://serrintine.github.io/StreamDoc/img/nanoconfbottom.png "nano")

Then add this:
```
rtmp {
  server {
    listen  1935;
    chunk_size 4096;

    application live {
      live on;
      record off;
    }
  }
}
```
![nano](https://serrintine.github.io/StreamDoc/img/editedconf.png "nano")
Save the file with Ctrl+O, then exit Nano with Ctrl+X.

Finally start the Nginx server:
```
sudo /usr/local/nginx/sbin/nginx
```

If you ever need to edit the file again, you'll need to reload Nginx for the changes to take effect:
```
sudo /usr/local/nginx/sbin/nginx -s reload
```
People already using your server won't be affected until their connection closes.

## Setup OBS
