# NPM_to_NGINX

A Tutorial Repo for migrating your Nginx Proxy Manager proxy setup to Nginx.

## Goal

To give clear instructions to help users migrate from using [Nginx Proxy Manager](https://nginxproxymanager.com/) (NPM) to standard [Nginx](https://docs.nginx.com/). This tutorial is not exhaustive and there are many other implementations of this transition. I would recommend checking out the many Nginx [Documentation Sites](http://nginx.org/en/docs/) and tutorials to learn more.

### Introduction

If you're anything like me and you got into the self-hosted/homelab/diy game sometime within the last 5 years, you've likely been recommended to use Nginx Proxy Manager as one of the choice Reverse Proxy services. If you've also been paying attention to various self-hosted communities, you may have also come across Christian Lempa's Video on [trusting smaller self hosted projects and tools](https://youtu.be/uaixCKTaqY0).

*Spoilers:* He roasts NPM in his video and towards the end says he won't be using NPM anymore. He also, perhaps purposely, doesn't share which tool he will be migrating to.

Whether you follow Christian away from NPM or not, it dawned on me that while NPM is using a very trusted web server and reverse proxy under the hood, I hadn't taken the time to understand how an Nginx Config actually worked. Since NPM was already creating most of the files for Nginx, I got to reading through all the files and reworking them so that I could begin using Nginx without the NPM gui.

*Contributing: This is not all encompassing of Nginx possibilities. Including instructions for various installation methods, using OpenResty, and any other migrations or use cases would help the community. If you'd like to add in additional information on how to migrate from NPM to Nginx, that is welcome. Simply submit a PR with your steps.*

### TL;DR - Quick Steps

1. Stop both NPM and Nginx first.
    * `systemctl stop nginx`
    * `docker stop npm` (or whatever you've named the container).

2. Copy the following contents (including sub-directories) from the NPM `/data/nginx` directory to the Nginx `/etc/nginx` folder:

    * `proxy_hosts` >  `sites-available`
    * `conf.d` > `conf.d`
    * `snippets` > `snippets`
    * `custom_ssl` > `custom_ssl` (if applicable)

3. Edit each file in your `sites-available` directory and update the paths. Most will change from `/data/nginx/` to `/etc/nginx`.

4. Edit your `nginx.conf` file and ensure the following two paths are there:

    * `include /etc/nginx/conf.d/*.conf;` and `include /etc/nginx/sites-enabled/*;`

5. Symlink the proxy host files in `sites-available` to `sites-enabled`

    * `ln -s * ./sites-enabled`

6. Test your changes with `nginx -t`. Make appropriate changes if there are error messages.

And that's it! You can now start Nginx and check for any errors using `systemctl status nginx`.

### Pre-requisites & Assumptions

I am using an Ubuntu VM with NPM and it's db as a Docker Container while Nginx is installed natively on the machine. You don't have to use this setup exactly, but I am making a few assumptions as to what you should have access to before you begin. I am also using custom SSL certs, but theoretically, the transition should be the same when using Lets Encrypt.

I've added some example files to show before and after changes to this repo and outlined file trees below.

* You understand the basics of what a Reverse Proxy is doing and are sticking with some stock settings (like exposing port 80 an 443).
* You've installed [NPM](https://nginxproxymanager.com/setup/) and [Nginx](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/) using your preferred method.
* You have access to both NPMs file tree and Nginx's.
* If using NPM in docker, make sure you've mapped a local volume on the host to the container.
* My setup using docker-compose is the following: `/user/nginx/data:/data`.
* Know where your Nginx files are. If using docker, same as above, make sure your container directories are mapped to the host.
* For a linux install, they should be accessible at `/etc/nginx`.
* You know how to edit files at the command line using `nano`, `vi`, `vim`, `neovim`, `emacs`, or something else.

### Nginx Files

Nginx uses the `nginx.conf` file and within that file, it will include your proxy files. These exist under `./nginx/sites-enabled/`. In the main `nginx.conf` file, the line `include /etc/nginx/sites-enabled/*;` will bring in those files to the config file, making the proxies accessible.

### How to Transition - Detailed Version

1. Since NPM uses Nginx under the hood, they are both, by default, going to try and use ports 80 and 443 to serve up your apps and content. Turn off both systems.

    * Docker: `docker stop [app_container, db_container]`
    * Systemd: `systemctl stop nginx`

2. Copy your `proxy_host` (NPM) files to the `sites-available` (Nginx) folder.
`cp -r /user/nginx/data/nginx/proxy_hosts/* /etc/nginx/sites-available/`

3. Nginx doesn't really care what the files are called, but NPM numbers them based on the order in which you added them in the GUI. I find it better to rename them to what service they actually serve up for easier identification later.

4. Copy your `custom_ssl` folder from NPM to the `custom_ssl` folder in Nginx. See the following step for the various default paths in both systems.

    * `cp -r /user/nginx/data/custom_ssl/* /etc/nginx/custom_ssl/`

5. Copy the `conf.d` folder from NPM to the `conf.d` folder in Nginx. *Note: For some reason, not all of the files in
   the proxy files were actually in my `conf.d` directory. If you're missing any files, please download and/or copy and
   paste them from the [NPM Repo](https://github.com/NginxProxyManager/nginx-proxy-manager/tree/fa851b61da3fe3726d1a04c25e69d36e79edea2d/docker/rootfs/etc/nginx/conf.d/include)*

    * `cp -r /user/nginx/data/nginx/conf.d /etc/nginx/`

6. If you had any additional files included in the Advanced section of an NPM Proxy Host, make sure you copy them over. For my setup and this tutorial, they were all located in the `snippets` directory.

    * `cp -r /user/nginx/data/nginx/snippets/* /etc/nginx/snippets/`

7. There are a number of lines that need to be updated in each proxy configuration file to make them work with Nginx. I've placed additional comments in [`./proxy_host/npm_proxy.conf`](./proxy_host/npm_proxy.conf) file. The line changes are the following:

    1. Custom SSL path:

        * NPM path: `/data/custom_ssl...`
        * Nginx path: `/etc/nginx/custom_ssl...`

    2. conf.d:

        * The paths should remain the same. However, if you changed the path for `conf.d` in Nginx and differed in step 5, above, make sure you use the correct path.

    3. Access & Error Logs

        * NPM path: `/data/logs/...`
        * Nginx path: `/var/log/nginx/...`

8. Double Check all your paths! If this is your first time using Nginx, make sure every directory is correct! Save your work.
9. Navigate to the `nginx.conf` file which is located at `/etc/nginx/`. You can see the one I am using in this repo.
10. Make sure that you have the following two lines in the main conf file and that they are pointing to the appropriate directories in the nginx directory path.
    * `include /etc/nginx/conf.d/*.conf;` and `include /etc/nginx/sites-enabled/*;`
11. You'll notice that you ensured there was a `sites-enabled` directory in the configuration file, but you changed all your proxy host config files in `sites-available`! Good eye, all that's left is to symlink the files to `sites-enabled` so that nginx can start using them.
12. To symlink the available proxy files run the following command within the `sites-available` directory:
    * `ln -s * ./sites-enabled`
13. Once you're confident that you've done all the above correctly, you can test your setup using nginx command and flags. While in the directory with your `nginx.conf` file - usually `/etc/nginx` - run the following command: `nginx -t`.
14. If all is working as expected you should see the below output. If it returns any errors, fix them appropriately. It will usually tell you what line is throwing the error. In this case, there's a high likelihood that it will be path error.

```bash
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

And that's it! You can now restart your nginx service on the host and access all your sites just as if you were using Nginx Proxy Manager! Make sure you take a look at your logs and system's status should nginx fail to start.

### Additional Information/Appendix

#### File Trees for NPM (in container) and Nginx (on host)

*I did not expand every directory in these trees. Only the ones that are pertinent for reference in this tutorial.*

#### NGINX

```bash
├── conf.d
│   └── include
│       ├── assets.conf
│       ├── block-exploits.conf
│       ├── force-ssl.conf
│       ├── ip_ranges.conf
│       ├── proxy.conf
│       └── resolvers.conf
├── custom_ssl
│   ├── npm-1
│   │   ├── fullchain.pem
│   │   └── privkey.pem
│   ├── npm-2
│   │   ├── fullchain.pem
│   │   └── privkey.pem
│   └── npm-3
│       ├── fullchain.pem
│       └── privkey.pem
├── fastcgi.conf
├── fastcgi_params
├── koi-utf
├── koi-win
├── mime.types
├── modules-available
├── modules-enabled
├── nginx.conf
├── proxy_params
├── scgi_params
├── sites-available
│   ├── auth.conf
│   ├── bitwarden.conf
│   ├── codehub.conf
│   ├── default.backup
│   ├── files.conf
│   ├── notes.conf
│   ├── photos.conf
│   ├── rsmsn-root.conf
│   ├── wordle.conf
│   └── wordle-it.conf
├── sites-enabled
│   ├── auth.conf -> /etc/nginx/sites-available/auth.conf
│   ├── bitwarden.conf -> /etc/nginx/sites-available/bitwarden.conf
│   ├── codehub.conf -> /etc/nginx/sites-available/codehub.conf
│   ├── files.conf -> /etc/nginx/sites-available/files.conf
│   ├── notes.conf -> /etc/nginx/sites-available/notes.conf
│   ├── photos.conf -> /etc/nginx/sites-available/photos.conf
│   ├── wordle.conf -> /etc/nginx/sites-available/wordle.conf
│   └── wordle-it.conf -> /etc/nginx/sites-available/wordle-it.conf
├── snippets
│   ├── authelia-authrequest-basic.conf
│   ├── authelia-authrequest.conf
│   ├── authelia-authrequest-detect.conf
│   ├── authelia-location-basic.conf
│   ├── authelia-location.conf
│   ├── authelia-location-detect.conf
│   ├── fastcgi-php.conf
│   ├── proxy.conf
│   └── snakeoil.conf
├── uwsgi_params
└── win-utf
```

#### NPM

```bash
├── data
│   ├── access
│   ├── custom_ssl
│   │   ├── npm-1
│   │   │   ├── fullchain.pem
│   │   │   └── privkey.pem
│   │   ├── npm-2
│   │   │   ├── fullchain.pem
│   │   │   └── privkey.pem
│   │   └── npm-3
│   │       ├── fullchain.pem
│   │       └── privkey.pem
│   ├── keys.json
│   ├── letsencrypt-acme-challenge
│   ├── logs
│   ├── mysql
│   └── nginx
│       ├── custom
│       ├── dead_host
│       ├── default_host
│       ├── default_www
│       ├── dummycert.pem
│       ├── dummykey.pem
│       ├── proxy_host
│       │   ├── 10.conf
│       │   ├── 11.conf
│       │   ├── 12.conf
│       │   ├── 13.conf
│       │   ├── 15.conf
│       │   ├── 1.conf
│       │   ├── 2.conf
│       │   ├── 4.conf
│       │   ├── 5.conf
│       │   └── 6.conf
│       ├── redirection_host
│       ├── snippets
│       │   ├── authelia-authrequest-basic.conf
│       │   ├── authelia-authrequest.conf
│       │   ├── authelia-authrequest-detect.conf
│       │   ├── authelia-location-basic.conf
│       │   ├── authelia-location.conf
│       │   ├── authelia-location-detect.conf
│       │   └── proxy.conf
│       ├── stream
│       └── temp
├── docker-compose.yml
└── letsencrypt
└── renewal-hooks
```

## Contact

You can me on Mastodon [@notnorm@fossotodon.org](https://fosstodon.org/@notnorm)
