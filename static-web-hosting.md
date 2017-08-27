# Static Web Hosting
This document describes how I host static websites 

## Important Directories / Files

- **`/etc/nginx/`:** Everything related to Nginx configuration lives in this folder.
- **`/etc/nginx/nginx.conf`:** The basic Nginx configurations.
- **`/etc/nginx/sites-available`:** A folder full of Nginx configuration files that may or may not be enabled.
- **`/etc/nginx/sites-enabled`:** A list of symlinks to files in `/etc/nginx/sites-available`.  The configuration files in this folder are active in Nginx.  To disable a configuration file, delete the symlink in this folder.
- **`/srv/`:** The root folder for all of the server's websites.
- **`/srv/schlosser.io`:** The folder for a particular website, in this case [schlosser.io][schlosser].
- **`/srv/schlosser.io/logs`:** The folder where all of a [schlosser.io][schlosser]'s logs should go.
- **`/srv/schlosser.io/logs/access.log`:** Every request sent to [schlosser.io][schlosser].
- **`/srv/schlosser.io/logs/error.log`:** Every request sent to [schlosser.io][schlosser] that resulted in an error.
- **`/srv/schlosser.io/public_html`:** The **LIVE** static content being served at [schlosser.io][schlosser].
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

## Deploying using Gulp

_Requires the "Setting up SSH key access to your sever" section to be complete._

In your `Gulpfile.js`, add a `deploy` task that copies the generated static content onto the server.

**Example for the [Minimill Project template](https://github.com/minimill/project-template):**

```js
var shell = require('gulp-shell');

gulp.task('deploy', ['build:optimized'], function() {
  gulp.src('')
    .pipe(shell('scp -r dist/* my-website:/srv/schlosser.io/public_html/'))
    .on('finish', function() {
      process.stdout.write('Deployed to schlosser.io\n');
    });
});
```

> **Reminder:** Be sure to replace `schlosser.io` with your website name and `my-website` with the alias you created in the previous section, in your `~/.ssh/config`.

**Example for [Jekyll](https://github.com/minimill/project-template/tree/jekyll):**

```js
var shell = require('gulp-shell');

gulp.task('deploy', ['build:optimized'], function() {
  return gulp.src('')
    .pipe(shell('scp -r _site/* my-website:/srv/schlosser.io/public_html/'))
    .on('finish', function() {
      process.stdout.write('Deployed to schlosser.io\n');
    });
});
```

> **Reminder:** Be sure to replace `schlosser.io` with your website name and `my-website` with the alias you created in the previous section, in your `~/.ssh/config`.

You should be all setup!  Try it out with `gulp deploy`.  This will use your SSH credentials to securely copy the generated static content into the appropriate `public_html` folder on the server.

## Adding a new domain / subdomain to the server

_This example is for adding a new subdomain, new.schlosser.io._

1. If this is a new domain, or if you haven't done so already, make sure that your DNS (Cloudflare, Namecheap, etc) is setup with an A record pointing your domain / subdomain to the IP address of the server.  

    > For ease of use, I redirect all subdomains to my server by using a wildcard (`*`) redirect.

2. Create the necessary folders in `/srv/`:

    ```bash
    localmachine:~$ ssh my-website
    my-website:~$ mkdir /srv/new.schlosser.io
    my-website:~$ mkdir /srv/new.schlosser.io/logs
    my-website:~$ mkdir /srv/new.schlosser.io/public_html
    ```

3. Create a new Nginx configuration file in `/etc/nginx/sites-available`
    
    ```bash
    my-website:~$ cd /etc/nginx/sites-available
    # Create the file. I use the format full.website.domain.nginx.conf, but
    # it's just a convention.
    my-website:/etc/nginx/sites-available$ touch new.schlosser.io.nginx.conf
    ```

    > **Tip:** You can skip this step if you want to just edit an existing configuration file.

4. Edit it (with sudo!) to add the necessary server blocks.

    **Example for a basic static site served over HTTP:**
    ```nginx
    server {
        listen 80;
        server_name new.schlosser.io;
        root /srv/new.schlosser.io/public_html;
        access_log /srv/new.schlosser.io/logs/access.log;
        error_log /srv/new.schlosser.io/logs/error.log;

        location / {
            index index.html;
        }
    }

    server {
        listen 443;
        server_name new.schlosser.io;
        rewrite ^ http://new.schlosser.io$request_uri? permanent;
    }
    ```

5. Enable the site by creating a symlink in `/etc/nginx/sites-enabled` to the configuration file you just created:

    ```bash
    my-website:/etc/nginx/sites-available$ cd /etc/nginx/sites-enabled
    my-website:/etc/nginx/sites-enabled$ ln -s /etc/nginx/sites-enabled/new.schlosser.io.nginx.conf new.schlosser.io.nginx.conf 
    ```

    > **Tip:** You can skip this step if you want to just edit an existing configuration file.

6. Test your Nginx configuration to ensure it's properly written. Ensure that you get a message as follows: 

    ```bash
    my-website:~$ nginx -t 
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

7. Restart Nginx to apply these configuration changes.

    ```bash
    my-website:~$ sudo service nginx restart
    ```

8. Push content to your `public_html` folder, using the instruction in the above section

9. You're done! See your changes live.


[schlosser]: https://schlosser.io
