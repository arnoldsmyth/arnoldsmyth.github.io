# LA~~M~~P on Mac with Homebrew

This is a guide I wrote so that I can set up an Apache, PHP server for local development. I don't use local MySQL so I have not included this. 

The end result will allow me to access my development sites at domains like `domain.lo` or `sub.domain.lo` and will have self signed certs for each domain.

I am aware that there may be some parts of this that can be done more efficiently - this was written as I figured some things out and needed to get it up and running and be able to be reproduced for future use.

This guide assumes a folder `/Users/YOUR_USERNAME/ApacheServer/` for all your files. You can update this to match your own location.

## Apache

**INSTALLATION**

1. Stop and unload the built in Apache by running the following two commands
`sudo apachectl stop`
`sudo launchctl unload -w /System/Library/LaunchDaemons/org.apache.httpd.plist 2>/dev/null`
2. Install httpd (Apache) with Homebrew
`brew install httpd`
3. Start background service and start Apache
`brew services start httpd`
`sudo apachectl start`
4. When opening now http://localhost:8080/ you should see “It works!”
5. Next get your username with `whoami` - you will need this throughout the process

**CONFIGURATION**

Open the file `/opt/homebrew/etc/httpd/httpd.conf` and change the following

1. Listen 8080 -> `Listen 80`
2. The path for DocumentRoot and Directory "/usr/local/var/www" -> `DocumentRoot /Users/YOUR_USERNAME/ApacheServer`
[All sites must be in or below this folder - put them in the format domain.lo for consistency]
3. Inside the previously edited “<Directory “/Users/YOUR_USERNAME/Sites”>” change AllowOverride None -> `AllowOverride All`
4. Uncomment #LoadModule rewrite_module lib/httpd/modules/mod_rewrite.so
5. Uncomment #LoadModule vhost_alias_module lib/httpd/modules/mod_vhost_alias.so
6. Uncomment #Include /usr/local/etc/httpd/extra/httpd-vhosts.conf
7. Uncomment #Include /opt/homebrew/etc/httpd/extra/httpd-vhosts.conf
8. User _www -> `User YOUR_USERNAME`
9. Group _www -> `Group staff`
10. Uncomment #ServerName www.example.com:8080 and change it to `localhost:80`
11. Open `/opt/homebrew/etc/httpd/extra/httpd-vhosts.conf`
12. Remove the existing virtual hosts
13. replace with the option you used above IncludeOptional IncludeOptional `/Users/YOUR_USERNAME/ApacheServer/*/httpd-vhost.conf`
14. In your webroot `/Users/YOUR_USERNAME/ApacheServer/` create a folder for each site like `domain1.lo` and `domain2.lo`
15. Create a folder in the root of each site called `logs`
16. Create a file in the root of each site you want to serve named `httpd-vhost.conf`
17. Add the following to the file created above:
`<VirtualHost *:80>`
`DocumentRoot "/Users/YOUR_USERNAME/ApacheServer/domain.lo"`
`ServerName domain.lo`
`ErrorLog "/Users/YOUR_USERNAME/ApacheServer/domain.lo/logs/error_log"`
`CustomLog "/Users/YOUR_USERNAME/ApacheServer/domain.lo/logs/access_log" common`
`</VirtualHost>`
12. Open `/private/etc/hosts`
13. Add to the end: `127.0.0.1 domain.lo`
15. Restart apache server with `sudo apachectl restart`.
16. Open http://localhost and should see the documents from your Sites directory. 

## PHP

**INSTALLATION**

1. Install the php formula with Homebrew.
`brew install php`
2. Start PHP background service
`brew services start php`

**CONFIGURATION**
Open the file `/opt/homebrew/etc/httpd/httpd.conf` and change the following

1. Insert after the last LoadModule (probably mod_rewrite.so):
`LoadModule php_module /opt/homebrew/opt/php/lib/httpd/modules/libphp.so`
2. DirectoryIndex index.html -> `DirectoryIndex index.html index.php`
3. Insert after the previously edited “DirectoryIndex” area and before the htaccess/htpasswd “IfModule” area:
`<FilesMatch \.php$>`
`SetHandler application/x-httpd-php`
`</FilesMatch>`
4. Restart your Apache server `sudo apachectl restart`

## SSL CERT
Also see the [Quick Access](#quick-access-ssl-commands)
**CONFIGURATION**
1. Create directory `ssl` in `/opt/homebrew/etc/httpd/` if not there
2. Create the conf file `sudo touch /opt/homebrew/etc/httpd/ssl/domain.lo.conf`
3. Add this to the conf you just created:

    `[req]`
    `default_bits = 1024`
    `distinguished_name = req_distinguished_name`
    `req_extensions = v3_req`
    
    `[req_distinguished_name]`
    
    `[v3_req]`
    `basicConstraints = CA:FALSE`
    `keyUsage = nonRepudiation, digitalSignature, keyEncipherment`
    `subjectAltName = @alt_names`
    
    `[alt_names]`
    `DNS.1 = domain.lo`

3. Change `domain.lo` to the correct domain and then Run the following commands
`sudo openssl genrsa -out /opt/homebrew/etc/httpd/server.key 2048`
`sudo openssl genrsa -out /opt/homebrew/etc/httpd/ssl/domain.lo.key 2048`
`sudo openssl rsa -in /opt/homebrew/etc/httpd/ssl/domain.lo.key -out /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa`
4. `sudo openssl genrsa -out /opt/homebrew/etc/httpd/server.key 2048`
5. `sudo openssl genrsa -out /opt/homebrew/etc/httpd/ssl/domain.lo.key 2048`
6. `sudo openssl rsa -in /opt/homebrew/etc/httpd/ssl/domain.lo.key -out /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa`
7. `sudo openssl req -new -key /opt/homebrew/etc/httpd/server.key -subj "/C=us/ST=wy/L=Cheyenne/O=Assessify/CN=domain.lo/emailAddress=webmaster@localhost/" -out /opt/homebrew/etc/httpd/server.csr`
8. `sudo openssl req -new -key /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa -subj "/C=/ST=/L=/O=/CN=domain.lo/" -out /opt/homebrew/etc/httpd/ssl/domain.lo.csr -config /opt/homebrew/etc/httpd/ssl/domain.lo.conf`

**Note**: Complete the values  `C= ST= L= O= CN=`  to reflect your own organizational structure, where:
-   `C=`  eq. Country: The two-letter ISO abbreviation for your country.
-   `ST=`  eq. State or Province: The state or province where your organization is legally located.
-   `L=`  eq. City or Locality: The city where your organization is legally located.
-   `O=`  eq. Organization: he exact legal name of your organization.
-   `CN=`  eq. Common Name: The fully qualified domain name for your web server
  
9. `sudo openssl x509 -req -days 365 -in /opt/homebrew/etc/httpd/server.csr -signkey /opt/homebrew/etc/httpd/server.key -out /opt/homebrew/etc/httpd/server.crt`
10. `sudo openssl x509 -req -extensions v3_req -days 365 -in /opt/homebrew/etc/httpd/ssl/domain.lo.csr -signkey /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa -out /opt/homebrew/etc/httpd/ssl/domain.lo.crt -extfile /opt/homebrew/etc/httpd/ssl/domain.lo.conf`
11. Add the cert to the Keychain so it will be trusted `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /opt/homebrew/etc/httpd/ssl/domain.lo.crt`

## Quick Access SSL Commands

Copy the code below to your favourite IDE and replace domain.lo with the domain you want to use. 

**Step 1**
Enter the following command:
`sudo cat > /opt/homebrew/etc/httpd/ssl/domain.lo.conf`

**Step 2**
Paste the config file (without the ##### lines) followed by `CTRL+D` to save the file.

######CONFIG FILE######

    [req]
    default_bits = 1024
    distinguished_name = req_distinguished_name
    req_extensions = v3_req
    
    [req_distinguished_name]
    
    [v3_req]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    
    [alt_names]
    DNS.1 = domain.lo

######END CONFIG FILE######

**Step 3**
Paste the following commands

    sudo openssl genrsa -out /opt/homebrew/etc/httpd/server.key 2048
    sudo openssl genrsa -out /opt/homebrew/etc/httpd/ssl/domain.lo.key 2048
    sudo openssl rsa -in /opt/homebrew/etc/httpd/ssl/domain.lo.key -out /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa
    sudo openssl req -new -key /opt/homebrew/etc/httpd/server.key -subj "/C=us/ST=wy/L=Cheyenne/O=Assessify/CN=domain.lo/emailAddress=webmaster@localhost/" -out /opt/homebrew/etc/httpd/server.csr
    sudo openssl req -new -key /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa -subj "/C=us/ST=wy/L=Cheyenne/O=Assessify/CN=domain.lo/" -out /opt/homebrew/etc/httpd/ssl/domain.lo.csr -config /opt/homebrew/etc/httpd/ssl/domain.lo.conf
    sudo openssl x509 -req -days 365 -in /opt/homebrew/etc/httpd/server.csr -signkey /opt/homebrew/etc/httpd/server.key -out /opt/homebrew/etc/httpd/server.crt
    sudo openssl x509 -req -extensions v3_req -days 365 -in /opt/homebrew/etc/httpd/ssl/domain.lo.csr -signkey /opt/homebrew/etc/httpd/ssl/domain.lo.key.rsa -out /opt/homebrew/etc/httpd/ssl/domain.lo.crt -extfile /opt/homebrew/etc/httpd/ssl/domain.lo.conf
    sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /opt/homebrew/etc/httpd/ssl/domain.lo.crt
