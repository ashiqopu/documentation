## Mini guide for TruaNAS Core Nextcloud remote access with Let's Encrypt and without port 80

There are usually four steps to set up `https` remote access to your Nextcloud TrueNAS jail. The first three are very straightforward and guides can be easily found online. The last step is the most confusing one as there is no guide that explicitly talks about `TrueNAS + Nextcloud + Let's Encrypt without port 80`.

*Please read through the full guide, see the related links carefully, that way you should understand what changes I had to make personally for the setup to work*.

#### Step 1: Router port forwarding.
We need to configure our router firewall to allow port forwarding to internal port `443` (for `https`) to our nextcloud jail's IP address (not TrueNAS ip address, if different). 

#### Step 2: [DuckDNS](https://www.duckdns.org/) for setting up a free Dynamic DNS (DDNS).
1. Go to their website and create a free account.
2. Create a Dynamic DNS of our choice in the form of `example.duckdns.org`.
3. Take a note of the DuckDNS token that's given to us, we will need that in next step.

#### Step 3: Setting up DDNS in TrueNAS.
[A simple guide can be found here](https://www.truenas.com/community/threads/how-to-install-duckdns-org-a-how-to-guide.24170/). You might find the Cron job creation option in a different TrueNAS menu deppending on your version.

#### Step 4: Setting up Let's Encrypt and remote access to Nextcloud jail in TrueNAS.

These steps are taken from a few sources, corrected/edited and combined together accordingly [[1](https://www.truenas.com/community/threads/enable-lets-encrypt-ssl-in-nextcloud-on-freenas.78734/)], [[2](https://jmorahan.net/articles/lets-encrypt-without-port-80/)].

1. We first `ssh` into our TrueNAS machine as root user from another machine:
    ```bash
    ssh root@ip_of_your_truenas
    ```

2. Within the session, `ssh` into our nextcloud jail
    ```bash
    iocage console nextcloud
    ```
    Note that `nextcloud` here should be the jail name, so if our jail name is `Nextcloud`, then we will use that, it is case sensitive.

3. Install nano text editor so we can edit a few config files.
    ```bash
    pkg update -f
    portsnap fetch extract
    cd /usr/ports/editors/nano/ && make install clean BATCH=yes
    ```

4. Edit `nginx.conf` to add our Dynamic DNS (many also refer to as FQDN) from DuckDNS.
    ```bash
    nano /usr/local/etc/nginx/nginx.conf
    ```
    Then add and change `example.duckdns.org` to our FQDN name we created earlier. Make sure to place the code inside the `http{...}` block, otherwise it will raise errors:
    ```bash
    server {
        listen 80;
        listen [::]:80;
        server_name example.duckdns.org;
        return 301 https://$server_name$request_uri;
    }
    ```
    Be sure to save the file when finished. This step enforces https.

5. Restart the nextcloud jail from your TrueNAS webGUI, then log back into nextcloud `ssh`.

6. This is where we will install and use [acme.sh](https://github.com/acmesh-official/acme.sh) to set up our Let's Encrypt certificate using `TLS-ALPN-01`, which only requires port 443. The guide can be found here, [Let's Encrypt without port 80](https://jmorahan.net/articles/lets-encrypt-without-port-80/). As our nextcloud jail is using nginx, we need to issue the certificate with `service` option:
    ```bash
    acme.sh --issue --alpn --pre-hook 'service nginx stop' --post-hook 'service nginx start' -d example.duckdns.org -w /home/wwwroot/example.duckdns.org
    ```

    After we issue the certificate, we need to install it as well in an appropriately secured directory (note we are using `wheel` group, avoiding to create root group),

    ```bash
    mkdir /etc/acme.sh
    chown root:wheel /etc/acme.sh
    chmod og-rwx /etc/acme.sh
    acme.sh --install-cert -d example.duckdns.org --key-file /etc/acme.sh/example.duckdns.org-key.pem --fullchain-file /etc/acme.sh/example.duckdns.org-cert.pem
    ```

    The last command for cert installation creates two files,
    a) `ssl_certificate`: should be in `/etc/acme.sh/example.duckdns.org-cert.pem`, and
    b) `ssl_certificate_key`: should be in `/etc/acme.sh/example.duckdns.org-key.pem`.
    We will need these two information in the next step. 

7. Copy the locations of the cert and key. Edit `nextcloud.conf` to enforce HTTPS
    ```bash
    nano /usr/local/etc/nginx/conf.d/nextcloud.conf
    ```
    Change the DDNS settings within the `server{...}` block:
    ```bash
    server {                       
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.duckdns.org;
        ssl_certificate /etc/acme.sh/example.duckdns.org-cert.pem;
        ssl_certificate_key /etc/acme.sh/example.duckdns.org-key.pem;
        add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; always;";
        ...
        ...
    }
    ```
    Be careful not to use `preload` in the `add_header` line. Why? Well, the warning and explanation is in the conf file itself,
    ```bash
    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    ```
    Be sure to save the file when finished.

8. We will now add our new DDNS to nextcloud trusted domains.
    ```bash
    nano /usr/local/www/nextcloud/config/config.php
    ```
    then add if needed,

    ```bash
    1 => 'example.duckdns.org',
    ```
    One of the previous lines should read,
    ```bash
    0 => localhost,
    ```

    You could also remove the local IP (which is usually option `1 => ip.to.your.nextcloud,`) and just have your DDNS name here. Be sure to save the file when finished.

12. Restart the nextcloud jail from your TrueNAS webGUI.

13. Test by going to your DDNS address example.duckdns.org.

Enjoy! :)