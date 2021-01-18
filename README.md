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
      - SSH into the VM
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
           `printf( __( 'Proudly powered by %s<br>%s', 'twentyseventeen' ), 'WordPress', $_SERVER['SERVER_ADDR'] );`
     - Login to phpMyAdmin 
         - ExternalIP/phpmyadmin 
		   - username: root
		   - password: mysql-root-password found on VM info page
         - Open the Wordpress database
         - Export the database: 
			   - Custom Export
			   - SQL format  
         - Upload exported db to your storage bucket
         - Create a new Cloud SQL Instance:
            - SQL > Create Instance > Choose MySql
            - Show Configuration options
	    - Under connectivity, enable Private IP
            - Under Backups, select High availability under Availability section
         - Create a new database
			   - Databases > Create Database
         - Import the SQL backup you uploaded to your bucket 
			   - Overview > Import 
			   - Select your bucket & backup
			   - Select your newly created database in the drop down
         - Create a new user
		      - Users > Add User Account
		      - Keep the username and password handy
		      - Allow any host
         - Edit the wp-config.php file, and change the following to their relevant details:
	 
	 	```
	 	define( 'DB_NAME', 'Your SQL Database Name' );
	 	define( 'DB_USER', 'Your SQL Database Username' );
	 	define( 'DB_PASSWORD', 'Your SQL Database User Password' );
	 	define( 'DB_HOST', 'Private IP Address of SQL Instance' );
	 	define( 'DB_CHARSET', 'utf8mb4' );
	 	define( 'DB_COLLATE', 'utf8mb4_general_ci' );
	 	```

	    
