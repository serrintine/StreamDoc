# Table of Contents
1. Setup AWS account
2. Create an EC2 instance
3. Edit security group
4. Install Bash on Ubuntu on Windows 10 (or Putty)
5. Setup RTMP server
6. Edit config file
7. Setup custom OBS stream
8. Play from VLC

## Setup AWS account
Go to [https://aws.amazon.com/free/](https://aws.amazon.com/free/) to sign up for a free AWS account.

![AWS free tier page](https://serrintine.github.io/StreamDoc/img/awsfreetier.png "AWS free tier page")

You will need to complete registration by filling in an extra form, which should be sent to your email. You will need to enter credit card info and receive a phone call for verification purposes, but you should not be charged any payment.

## Create an EC2 instance
Once your account is running you should be able to see a dashboard of all the services AWS provides. Scroll down to **Build a solution** and click on "Launch a virtual machine" to start the steps for creating a free EC2 instance.
![Build a solution](https://serrintine.github.io/StreamDoc/img/buildasolution.png "Build a solution")

Choose **Ubuntu** as the server OS. You will by default be alloted the free tier server configuration, which is a **t2.micro** instance with 1GB memory and 8GB storage.

You will be prompted to download a private key (a `.pem` file) to your server. Make sure you save it somewhere safe on your computer and do NOT share it with anyone as it is essentially the password to your server.

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

First get an updated list of software you can get through the package manager. Sudo runs a command as root (all the privileges), and we need those privileges. You'll need Mercurial (to get the Nginx sources) and Git (for the RTMP module sources). The other things in the list are software packages that Nginx needs.

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

Then add this at the end of the file:
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
Note that you can right click to paste, like this:

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

## Setup custom OBS stream
In Settings -> Stream, select "Custom Streaming Server" for stream type. The URL will be rtmp://server/live, where server is your server. If you're using Amazon EC2, use the Public DNS there.

Pick anything for the stream key. You'll use it later when getting VLC to play the stream.
![obs server setting](https://serrintine.github.io/StreamDoc/img/obsserver.png "obs server setting")

Then, start streaming as you would with Twitch, by hitting the "Start Streaming" button.

Tweaking OBS to make your stream smooth is a lot like handling surprise Rakkhat aggro. The best way forward is to carefully ponder your options while staying calm. 

For all of the encoding options, there are some common fields:
- Bitrate - How much internet upload bandwidth your stream will use. This value is typically in kbps. Entering 4500 will consume about 4.5 Mbps of upload bandwidth. You don't want to push this anywhere near what your internet provider claims it can handle.
- Rate Control - You have limited internet bandwidth to upload video, and this option controls how OBS makes quality compromises.
    - VBR - Variable Bit Rate. Consumed upload bandwidth will hover around the bitrate you give, but can go up or down depending on how much motion there is in the video. Probably the best option.
    - CBR - Constant Bit Rate. Consumed upload bandwidth will stay at the bitrate you give. If there's more motion, the encoder will drop quality rather than eat more bandwidth, and if there's not much motion, the encoder will eat bandwidth it doesn't need.
    - CRF - Constant Rate Factor - you don't specify a bitrate. You ask for a quality (lower is better, 23 is a good default), and the encoder eats all the bandwidth it needs to get there. If there's a lot of motion (i.e. Rakkhat has given you a beach ball and you're running in circles screaming), OBS will eat all the bandwidth required to describe that. Good luck.
- Keyframe Interval - Video encoding saves upload bandwidth by not uploading every frame. Instead, a frame is uploaded, and then the next frames are described in terms of how they differ from the last frame. A keyframe is a complete frame. I've found that auto causes some problems, even though it should be good in theory. A keyframe interval of 3 seconds worked well. 

The higher you can set your bitrate without killing your internet connection, the better quality gets. Using a good encoder helps you get better quality at a given bitrate.

Encoder options:
* x264 software encoder - You can consider encoding the video with your CPU if you have plenty of CPU cores (more than four is a good rule). AMD's Ryzen 7 and higher end Ryzen 5 chips are good candidates for this. Intel's i7-6800K, i7-5820K, and upcoming i7-8700K should also do well. Using the CPU gives the best quality.
![OBS software encoding](https://serrintine.github.io/StreamDoc/img/obsenc1.png "OBS software encoding")
* Intel QuickSync - If you have an Intel CPU with integrated graphics, like the i7-4970 or i5-6600K, this is an option. Intel designed circuitry that can encode video into most of their consumer CPUs. Using that frees up CPU cores for other tasks, like running ESO. It's very fast, but gives lower quality than CPU encoding.
![OBS qsv encoding](https://serrintine.github.io/StreamDoc/img/obsqsv.png "OBS qsv encoding")
* Nvidia NVENC - If you have a midrange to high end Nvidia GPU from the GTX 600 series or higher, your GPU has video encoding circuitry too. Much like Intel QuickSync, it's fast and doesn't demand much from your CPU, but looks worse than x264 CPU encoding.
![OBS nvenc encoding](https://serrintine.github.io/StreamDoc/img/obsnvenc.png "OBS nvenc encoding")
* AMD VCE - If you have an AMD GPU that's newer than the HD 7000 series, you can use AMD VCE. It's pretty bad but may have gotten better, details to follow. 

## Play from VLC
Open VLC. Select "Open Network Stream":

![open vlc stream](https://serrintine.github.io/StreamDoc/img/vlcopenstream.png "open vlc stream")

In the box that pops up, enter the server URL from above along with the stream key:

![vlc open stream](https://serrintine.github.io/StreamDoc/img/vlcopenstream1.png "vlc open stream")
