# cloud-1

###### Introduction to databases, data and cloud servers

## Project Requirements:

Install a simple wordpress site on cloud infrastructure.

   - Website must be running on at least 2 servers (if possible, in different server farms).
   - A Load Balancer to redirect requests to any of the available servers.
   - Traffic spikes will cause new servers to launch, and traffic decreases will cause new servers to shutdown.
   - Logged in users will stay identified for the length of a normal session.
   - New content on the site should be available across all instances as soon as possible.
   - Static content should be distributed via CDN.
   - Failures should be handled so that website uptime is not interrupted.
   - Hosting cost should reflect actual usage.
   - Appropriate firewall rules / protection measures should be in place as to avoid inappropriate access.

## Google Cloud Platform

The following steps were taken to set up this project:

   1. Create an account on GCP (possibly eligible for $300 free credit for 90 days)
   2. Create a new project
   3. Create a Wordpress Virtual Machine (VM)
      - Compute Engine > VM instances > Create an instance
      - Select Marketplace
      - Search for Wordpress (Google Click to Deploy) & Launch
	   - Use a region closest to you
	   - Add admin email 
   4. Create a new Service Account
      - IAM & Admin > Service Accounts > Create Service Account
      - Assign roles:
        	
		- `Cloud SQL > Cloud SQL Client`
		- `Project > Editor`
      - Create a key for the service account and download in JSON format
   5. Reserve a static IP
      - VPC Network > External addresses > Reserve Static Address
		   - Premium tier
		   - IPv4
		   - Global
   6. Create a Storage Bucket
      - Storage > Create Bucket
		- Multi-region
		- Location closest to you
   7. Configure Wordpress & the server
      
      i. SSH into the VM
         - Compute Engine > VM instances
         - Under the `connect` tab, click SSH
         - `sudo su`
         - `cd /var/www/html`
         - Create an .htaccess file & add the following:
         
            ```
            #RewriteEngine On
            #RewriteCond %{HTTP:X-Forwarded-Proto} =http
            #RewriteRule .* https://%{HTTP:Host}%{REQUEST_URI} [L,R=permanent]
            ```
         - Open wp-config.php file & add the following below the Database definitions:
         
            ```
            define( 'AS3CF_SETTINGS', serialize( array(
                  'provider' => 'gcp',
                  'key-file-path' => '/etc/keyFile.json',
            ) ) );
      
            #define('FORCE_SSL_ADMIN', true);
            #if (strpos($_SERVER['HTTP_X_FORWARDED_PROTO'], 'https') !== false)
            #    $_SERVER['HTTPS']='on';
            ```
         - `cd /etc/ && vim keyFile.json`
         - Add the contents of the previously downloaded service account key JSON file to keyFile.json
         - `cd /var/www/html/wp-content/themes/twentyseventeen/template-parts/footer`
         - open site-info.php file & adjust the following line to look as follows:
      
      		```
      		printf( __( 'Proudly powered by %s<br>%s', 'twentyseventeen' ), 'WordPress', $_SERVER['SERVER_ADDR'] );
      		```
		
      ii. Login to phpMyAdmin:
      	 - ExternalIP/phpmyadmin 
	     - username: root
	     - password: mysql-root-password found on VM info page
         - Open the Wordpress database
         - Export the database
	 		- Custom Export
			- SQL format
	     - Upload exported db to your storage bucket
         - Create a new Cloud SQL Instance (this may take a few minutes):
	 		- SQL > Create Instance > Choose MySql
            - Show Configuration options
			    - Under connectivity, enable Private IP
                - Under Backups, select High availability under Availability section
	    - Create a new database:
	 		- Databases > Create Database
	    - Import the SQL backup you uploaded to your bucket:
	 		- Overview > Import 
			- Select your bucket & backup
			- Select your newly created database in the drop down
		- Create a new user
		    - Users > Add User Account
			- Keep the username and password handy
			- Allow any host
		- Edit the wp-config.php file, and change the following to their relevant details:
	    	```
		    define( 'DB_NAME', 'Your Database Name' );
            define( 'DB_USER', 'Your SQL Database Username' );
            define( 'DB_PASSWORD', 'Your SQL Database User Password' );
            define( 'DB_HOST', 'Private IP Address of SQL Instance' );
            define( 'DB_CHARSET', 'utf8' );
            define( 'DB_COLLATE', 'utf8_general_ci' );
            ```
        iii. Login to Wordpress Admin:
        - ExternalIP/wp-admin, using the details on the VM info page.
        - Install & Activate the WP Offload Media Lite & W3 Total Cache plugins
        - In the WP Offload Media settings:
            - Browse existing buckets & select previously created Storage Bucket
        - In the W3 Total Cache settings:
            - In the CDN sub section of General Settings, Enable CDN and use Generic Mirror & save settings
            - Back at the top, Click on View Required Changes and add the text displayed to your .htaccess file, below the commented out lines already there.
        - ONLY ONCE YOU ARE SATISFIED WITH THE WORDPRESS CONFIGURATION:
            - Change the 2 site URL fields in the General Wordpress settings to either point to your domain (if you're using one) or the reserved static IP.
            
        iv.  IF USING A DOMAIN, uncomment the 3 lines in each of the .htaccess and wp-config.php files that deal with HTTPS.
    
8. Shut down your wordpress instance
9. Create a new custom image:
    - Compute Engine > Images > Create Image
	- Select VM as source disk
10. Create a new instance template:
    - Compute Engine > Instance Template > Create instance template
	- Change the boot disk to a custom image & select the image you just created
	- Allow HTTP / HTTPS traffic
11. Create new Firewall Rule:
    - VPC Network > Firewall > Create a new rule:
    - Ensure Direction is Ingress
    - Targets = All Instances
    - Source Filter = IP Ranges
    - Ranges = 130.211.0.0/22, 35.191.0.0/16
    - Specific TCP port of 80.
12. Create a new Managed Instance Group:
    - Compute Engine > Instance Groups > Create instance group
	- Select the Instance template you created
	- Autoscale
	- Min instances: 2 , Max instances: 6
	- Create health check (TCP) on port 80
## If using a Domain:
13. Network Services > Loadbalancing:
    i. HTTPS Loadbalancer:
    - Create a new HTTP/S Load balancer
    - From Internet to VMs
    - Create a new backend service:
        - HTTP protocol over port 80
        - Instance group = your instance group
        - Enable Cloud CDN
        - Apply your HTTP health check.
    - Create a new frontend service:
        - HTTPS protocol over port 443
        - Set the IP address as your previously reserved static IP address.
        - Create a new SSL certificate (Google Managed, use your domain name or Loadbalancer IP address [Not Tested myself with loadbalancer ip])
    - Finish

    ii.  HTTP Loadbalancer:
    - Create a new HTTP/S Load balancer
    - From Internet to VMs
    - Skip Backend Service
    - Host Path and Rules:
        - Advanced
        - Redirect to Different Host/Path
        - Prefix Redirect
        - Status 301
        - Enable HTTPS redirect
    - Create a new frontend service:
        - HTTP protocol over port 80
        - IP address is previously reserved static IP address.
    - Finish
## If NOT using Domain (we were not):
13. Create a Load Balancer:
    - Network Services > Load Balancing > Create load balancer     	
	- HTTP/S Load balancer
    - From Internet to VMs
    - Create a new backend service:
        - HTTP protocol over port 80
        - Instance group = your instance group
        - Enable Cloud CDN
        - Apply your HTTP health check.
    - Create a new frontend service:
        - HTTP protocol over port 80
        - IP address is previously reserved static IP address.
14. You may have to wait 10-15min for everything to start working together.
15. Once everything is setup, the Wordpress homepage should load when accessing the loadbalancer IP.

## Resources:
- https://themeisle.com/blog/install-wordpress-on-google-cloud/
- https://geekflare.com/google-cloud-cdn-test/
- https://geekflare.com/gcp-load-balancer/
- https://themeisle.com/blog/install-wordpress-on-google-cloud/
- https://www.cloudbooklet.com/best-performance-wordpress-with-google-cloud-cdn-and-load-balancing/
- http://www.mindinfinity.net/autoscaling-with-google-cloud/index.html
- https://cloud.google.com/compute/docs/instance-groups
- https://geekflare.com/google-cloud-sql-wordpress/

## Contributors
[amaquena](https://github.com/Amaquena) [@Deathshadowown](https://github.com/Deathshadowown)
## Final Mark 100/100
     
       
