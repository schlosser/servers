# Flask
This tutorial describes how I host Flask on a web server.

## A note about syntax in this tutorial:

When I instruct you to write a terminal command, I write them out in a very verbose way. Because we're dealing with both a server, a local machine, and a variety of folders, I use this syntax:

```bash
[machine-name]:[folder]$ [command]
```

For example, to get into the `/srv/` folder on your server, you might start by running the `ssh` command on your local machine, from the home directory:

```bash
localmachine:~$ ssh my-website
```

And then, changing directory to the `/srv/` folder once you're on the server:

```bash
my-website:~$ cd /srv/
```


## Important Directories / Files

- **`/etc/nginx/`:** Everything related to Nginx configuration lives in this folder.
- **`/etc/nginx/nginx.conf`:** The basic Nginx configurations.
- **`/etc/nginx/sites-available`:** A folder full of Nginx configuration files that may or may not be enabled.
- **`/etc/nginx/sites-enabled`:** A list of symlinks to files in `/etc/nginx/sites-available`.  The configuration files in this folder are active in Nginx.  To disable a configuration file, delete the symlink in this folder.
- **`/srv/`:** The root folder for all of the server's websites.
- **`/srv/mywebsite.com`:** The folder for a particular website, in this case mywebsite.com.
- **`/srv/mywebsite.com/logs`:** The folder where all of a mywebsite.com's logs should go.
- **`/srv/mywebsite.com/logs/access.log`:** Every request sent to mywebsite.com.
- **`/srv/mywebsite.com/logs/error.log`:** Every request sent to mywebsite.com that resulted in an error.
- **`/srv/mywebsite.com/www`:** The Flask code being run at mywebsite.com.
- **`~/bin/`:** Scripts that run on the server
- **`~/bin/deploy_all.sh`:** The script that stops all Flask servers, checks out the latest version from Git, and restarts all of them.
- **`~/git/`:** All the git repos on my server that I can push to.
- **`~/git/mywebsite.com.git`:** The git repo I push to for mywebsite.com.
- **`~/git/mywebsite.com.git/hooks/post-receive`:** The script that gets run every time I push to the git repo for mywebsite.com.
- **`~/.ssh/authorized_keys`:** The list of public keys of people who can SSH into the server.

## Setting up SSH key access to your server.

1. Follow [these instructions from DigitalOcean](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04#step-four-—-add-public-key-authentication-(recommended)) to generate an SSK key pair on your local machine, and add it to the `~/.ssh/authorized_keys`. If you've done this correctly, you should be able to SSH into the server without entering your password.

2. [Optional] Continue on to the [next section of that DigitalOcean tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04#step-five-—-disable-password-authentication-(recommended)) to disable password authentication for better security.

3. On your **local machine**, add an alias for your server to your SSH config file (you may have to make one), `~/.ssh/config`:

    ```
    Host my-website
      HostName 123.456.789.012
      User yourusername
      IdentityFile ~/.ssh/id_rsa
    ```

    > Make sure to replace `my-website` with a short, easy to type alias for your website (I use `dan`), `123.456.789.012` with the IP address of your server, and `yourusername` with the username of the unix user on your account (for me this was also `dan`).

## Deploying using Git

_Requires the "Setting up SSH key access to your sever" section to be complete._

On your local machine, in the git repo for your project, add a new git remote to push to. Assuming that `origin` is set to your GitHub URL, add a new origin called `deploy` that's pushed to your server:

```bash
localmachine:~/mywebsite$ git remote add deploy my-website:~/git/mywebsite.com.git
```

> Be sure to replace `my-website` the the server alias you chose in the first section, and `mywebsite.com.git` to the name of your git repo that you made while configuring the server.

That's it! You should be ready to push just using `git`:

```bash
localmachine:~/mywebsite$ git push deploy master
```

## Setup the server to run Flask the first time.

_Here, we'll be setting up a server, fresh. It will be configured for a fake website, mywebsite.com_

1. If this is a new domain, or if you haven't done so already, make sure that your DNS (Cloudflare, Namecheap, etc) is setup with an A record pointing your domain / subdomain to the IP address of the server.  

    > For ease of use, I redirect all subdomains to my server by using a wildcard (`*`) redirect.

2. Create the necessary folders in `/srv/`. These will hold the files related to the live, running server.

    ```bash
    localmachine:~$ ssh my-website
    my-website:~$ mkdir /srv/mywebsite.com
    my-website:~$ mkdir /srv/mywebsite.com/logs
    my-website:~$ mkdir /srv/mywebsite.com/www
    ```

3. Create a new Git repo that you'll push to from you local computer to deploy. Then, inside the Git repo, add a `post-recieve` hook, which is a bash script that will be run whenever you push to this repo.

    ```bash
    my-website:~$ mkdir git
    my-website:~$ cd git
    my-website:~/git$ git init --bare mywebsite.com.git
    my-website:~/git$ cd mywebsite.com.git/hooks
    my-website:~/git/mywebsite.com.git/hooks$ touch post-receive
    ```

4. Edit the `post-receive` hook, so that it runs a global "deploy-all" script.

    ```bash
    #! /bin/bash
 
    ~/bin/deploy.sh
    ```

5. Now, create and edit `deploy.sh`. It will stop all running servers, then restart them all again.

    ```bash
    #! /bin/bash

    ############## Repeat for each Flask Server you have ##############

    # Variables, two per site. You can imagine having a bunch of these.
    REPO_MYWEBSITE=~/git/mywebsite.com.git
    PUBLIC_WWW_MYWEBSITE=/srv/mywebsite.com/www

    ########################### End Repeat ############################ 

    # Stop all running flask apps
    pkill gunicorn

    # Pretty printouts are pretty!
    echo   "==============================================="
    echo   "Deployment Starting"
    echo   "-----------------------------------------------"
    echo   "Progress:"

    ############## Repeat for each Flask server you have ##############

    printf "Deploying mywebsite.com                  "

    # Go into the MYWEBSITE git repo, and checkout the latest code
    # into the code directory under /srv/
    cd "$REPO_MYWEBSITE"
    GIT_WORK_TREE=$PUBLIC_WWW_MYWEBSITE git checkout -f
    cd "$PUBLIC_WWW_MYWEBSITE"

    ##--## 
    ##--## App-specific Code: These next few lines may change depending on how
    ##--## you choose to run your Flask app. For me, I build with virtualenv
    ##--## and a requirements.txt. These commands below correspond with using
    ##--## the following commands on a local machine:
    ##--## 
    ##--##     localmachine:~/myrepo$ virtualenv .
    ##--##     localmachine:~/myrepo$ source bin/activate
    ##--##     localmachine:~/myrepo$ pip install requirements.txt
    ##--##     localmachine:~/myrepo$ python myapp.py
    ##--## 
    ##--## Yours may look different. In this example, the only real difference
    ##--## between the local and the server environments is installing 
    ##--## gunicorn, and the addition of the -q flags to run the commands 
    ##--## silently. Remove them to see everything that happens when you 
    ##--## deploy, add them back for pretty printing.
    ##--## 
    
    # Run the flask app! Here, we make a virtual environment
    virtualenv -q .

    # Then enter that virtual environment
    source bin/activate

    # Pip install requirements
    pip install -qr requirements.txt


    # Pip install gunicorn, which we’re using to run flask multi-threaded
    pip install -q gunicorn

    # The "myapp:app" is key here. This command hooks into "myapp.py" with 
    # gunicorn, and grabs the Flask object that’s called "app". E.g.: 
    #
    #   # myapp.py:
    #   app = Flask(__name__)
    #
    # We’re running this at port 8001 and sending the app logs to
    # /srv/mywebsite.com/logs/gunciorn.log and everything else 
    # to /dev/null
    #
    # The `nohup` command runs this in the background, so we can safely 
    # exit the server and keep Flask running.
    nohup \
        gunicorn myapp:app \
            -b 127.0.0.1:8001 \
            --log-file ../logs/gunicorn.log \
        nohup.out \
        2>&1 < /dev/null \
        &

    ##--##
    ##--## End App-specific code
    ##--##

    # exit virtual environment.
    deactivate
    printf "Done\n"

    ############################# End Repeat ############################# 

    # Now some silly code that checks if the site is actually live
    # (returns 200)
    echo   "-----------------------------------------------"
    echo   "Status:"

    ############## Repeat for each Flask Server you have ##############
    printf "mywebsite.com                             "
    if [[ -z "$(curl -Is http://mywebsite.com | head -1 | grep 200)" ]]; then
        printf "Fail\n"
    else
        printf "Live\n"
    fi
    ############################# End Repeat ############################# 
    
    echo   "----------------------------------"
    echo   "Deployment Complete"
    echo   "=================================="
    exit
    ```

    > In the future, you'll be able to SSH into the server and run this script to manually restart all of your Flask servers.

6. Now, if we were to push, we'd successfully be running the Flask app on localhost.  Next, we'll connect it to our domain name using Nginx. Create a new Nginx configuration file in `/etc/nginx/sites-available`:
    
    ```bash
    my-website:~$ cd /etc/nginx/sites-available
    # Create the file. I use the format full.website.domain.nginx.conf, but
    # it's just a convention.
    my-website:/etc/nginx/sites-available$ touch mywebsite.com.nginx.conf
    ```

7. Edit it (with sudo!) to add the necessary server blocks. This will redirect any traffic coming in at `http://mywebsite.com`, and redirect to the server running at `localhost:8001`.

    ```nginx
    server {
        listen 80;
        server_name mywebsite.com;

        root /srv/mywebsite.com/www;
        access_log /srv/mywebsite.com/logs/access.log;
        error_log /srv/mywebsite.com/logs/error.log;

        location / {
            proxy_pass http://127.0.0.1:8001;
            proxy_set_header        X-Real-IP       $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    ```

7. Enable the site by creating a symlink in `/etc/nginx/sites-enabled` to the configuration file you just created:

    ```bash
    my-website:/etc/nginx/sites-available$ cd /etc/nginx/sites-enabled
    my-website:/etc/nginx/sites-enabled$ ln -s /etc/nginx/sites-enabled/mywebsite.com.nginx.conf mywebsite.com.nginx.conf 
    ```

9. Test your Nginx configuration to ensure it's properly written. Ensure that you get a message as follows: 

    ```bash
    my-website:~$ nginx -t 
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

10. Restart Nginx to apply these configuration changes.

    ```bash
    my-website:~$ sudo service nginx restart
    ```

11. Push your Flask code to the Git repo.

12. You're done! See your changes live.
