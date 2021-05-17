+++
title = "Linking Bitbucket Server to Netlify"
tags = ["atlassian"]
date = 2020-03-27T00:00:00.000Z
layout = "blog"
description = "Using Bitbucket behind the firewall with tools like Netlify can be difficult to set up; Attomus provides a step by step guide"
+++
One of the challenges facing companies using [Atlassian](https://atlassian.com) products behind the firewall is how to configure them to get the same functionality offered by their Cloud brethren.  At Attomus we love [Netlify](https://netlify.com) but setting it up to auto-trigger builds from private repositories on [Bitbucket Server](https://www.atlassian.com/software/bitbucket) running behind a proxy and firewall - is not as simple as one would hope!

Having gone through the process ourselves, we thought it could be valuable for others to put together a quick step by step guide; in our example we've got an NginX server acting as a proxy facing the internet, with bitbucket running on a private server, and we want to both trigger builds automatically when we commit an update to the repository, and to ensure Netlify has visibility of the code in order to actually build our site.

*The same logic can be used for any scenario where we have a third party application that we want to integrate from the outside, Netlify is just an example.*

We're going to run through the following steps:

* Ensure Bitbucket can accept inbound SSH connections
* Configure NginX to route outside SSH connections through to our Bitbucket server
* Use the Netlify CLI to manually create our site and get the keys needed to link BitBucket to Netlify
* Update BitBucket to set the keys needed to gain access to our repository, and the webhook to trigger the build
* Test

## Open up Bitbucket for SSH connections

So the first thing we need to ensure is that we have set up Bitbucket to accept SSH connections for pushing code and accessing our repositories.  Usually (because it makes the config much easier!) we find that clients running behind a proxy like Apache or NginX only permit access via http, but Netlify requires SSH access so we're going to have to open things up a little.

Let's pop into our Bitbucket installation through the web front end, and go into Server Settings.   Scroll down to SSH access and ensure we have ticked 'SSH enabled', 'SSH access keys enabled' and then the details needed for your own installation.  Note that the 'SSH port' is the port that Bitbucket is listening on your local bitbucket server, and the 'SSH Base URL' is the proxied URL which is the one that NginX is going to be listening on from outside your firewall. So in our example, we are facing the internet via our proxy as `repo.MYSITE.com` but our internal IP address is `10.1.2.3`.

![Enabling SSH keys](/blog-media/bitbucket-enabling-ssh-access.png "Enabling SSH keys") 

Let's quickly check that bitbucket is listening by using the following command

```bash
    netstat -tulpn | grep bitbucket
```

and then we can set our proxy up to match.

*One thing to be aware of is that if you are using Bitbucket inside your firewall as well as from the outside, you'll need to ensure the /etc/hosts file on the Bitbucket server maps the external URL to your local IP address - e.g.* 

```bash
    127.0.0.1 localhost
    127.0.0.1 repo.MYSITE.com
```

## Set up NginX to proxy external SSH

So now we know that Bitbucket is listening for SSH which means those inside our firewall can access fine, but we need to ensure those outside (including Netlify) can get through to our repository, but without exposing our Bitbucket server directly to the internet.  

To achieve this, we have to firstly ensure that NginX can handle streams (ssh is effectively just a TCP data stream) and not just its normal http traffic, and then set up the proxy to forward SSH traffic from the internet to our NginX server to correctly talk to our Bitbucket server.

First of all, we need to make sure our installation of NginX can support streams.  On a Debian machine, this means we need to install NginX-extras

 

```bash
    sudo apt-get install nginx-extras
```

Then we need to update nginx.conf to include our config file, similar to how we probably already have with our website traffic in a server block under `/etc/nginx/sites-enabled/`.

```bash
       # put this at the bottom of your nginx.conf file
   
       # Pick up SSH proxies
       stream {
           # Reverse proxy config files
           include /etc/nginx/streams-enabled/*;
       }
```

Create the directory `/etc/nginx/streams-enabled/` and create a config file for your site `repo.MYSITE.com`

```bash
    upstream bitbucket-ssh {
        server 10.1.2.3:7999; # Bitbucket internal IP
    }
    server {
        listen 22222; # proxied listener
        proxy_pass bitbucket-ssh;
    }
```

Once that's saved we can quickly check that our config is correct using `nginx -T` where we should see that our new config file has been picked up. If so, we can restart with `service nginx restart` to pick up the new config, and double check it with 

```bash
    netstat -tulpn | grep 22222
```

To finish up the NginX config, we can check our config is working correctly by connecting from outside our firewall with our usual access details:

```bash
    ssh username@repo.MYSITE.com -p 22222
```

## Configure Netlify & Bitbucket to talk to each other

Next we need to create our Netlify site and get the public keys that will be used to authenticate with our Bitbucket server.

At the time of writing, Netlify only allows us to connect to Bitbucket Cloud, Github and GitLab, so we're going to have to use the CLI to link our private repository.  First we're going to have to install the cli tool and then link to our site :
```bash 
    npm install netlify-cli -g
    netlify sites:create --manual --with-ci
```

The `--manual` means that we're going to be wanting the keys to link Netlify and Bitbucket together, and `--with-ci` will give us the webhook so we can trigger the build automatically when code is pushed to our repository.
    
The CLI tool will prompt for a site name (you'll need to talk to Netlify support if you're wanting to link to an existing Netlify site, it's not possible at the moment from the command line), followed by the SSH remote URL - you can find this by going to your repository in Butbucket, clicking on Clone, then selecting the SSH option to find out the path.  It will be something like

```bash
    ssh://git@repo.MYSITE.com:22222/myproject/mycode.git
```

and you'll then be given a public key that you'll have to cut and paste into the Bitbucket repository settings under access keys:

Once this is keyed you can continue with the CLI tool, setting your build settings (which you can change later in the GUI) until you get provided with a webhook that needs to be cut and pasted into `Repository / Settings / Webhooks`.

## Test

That's it, we should be up and running now.  We've already checked that we have NginX proxying ok, so as soon as we completed our CLI tool Netlify would have connected in and, hopefully, built our site.  If you navigate to Netlify and check you should find your site already built (or at least, attempted - check the build logs!).  The final thing to check is the webhook is working, so push a small change to your repository and make sure that Netlify fires off a new build.

That's it, you're done!