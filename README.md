# My Cloud Computing Project: setting up a highly available and scalable WordPress website

## 1. Create separate Security Groups:
1. Create a SG for my ALB (Application Load Balancer) - `JeremProject_ALB-SG`:
    - Inbound rule: **HTTP (80) / HTTPS (443)** from Anywhere (allowing HTTP/S connections to the ALB from anywhere)
    - Outbound rule: **HTTP (80) / HTTPS (443)** to my EC2 SG (`JeremProject_EC2-SG`)

2.  Create a SG for my EC2 instances - `JeremProject_EC2-SG`:
    - Inbound rules: 
        - **SSH (22)** from Anywhere (allowing SSH via Instance Connect - especially for when working on my Windows environment where SSH is painful to use)
        - **HTTP (80) / HTTPS (443)** from my ALB SG (`JeremProject-ALB-SG`)
    - Outbound rule: All traffic / Everywhere

3. Create a SG for my EFS (Elastic File System) - `JeremProject_EFS-SG`:
    - Inbound rule: **NFS (2049)** from my EC2 SG (`JeremProject-EC2-SG`)
    - Outbound rule: All traffic / Everywhere

4. Create a SG for my RDS (Relational Database) - `JeremProject_RDS-SG`
    - Inbound rule: **MySQL (3306)** from my EC2 SG (`JeremProject_EC2-SG`)
    - Outbound rule: All traffic / Everywhere

 
## 2. Create a MySQL RDS:
- Standard Create -> MySQL -> v8.0.35 -> Free Tier template
    - Settings:
        - Identifier: `jeremproject-wordpress`
        - DB name: `wordpress`
        - Master username: `admin`
        - Master password: `1234567890`
    - Instance config: `db.t3.micro` (bc of Free Tier template)
    - Connectivity: 
        - "Don't connect to EC2"
        - Default VPC
        - Default subnet
        - NO public access
        - Security group: `JeremProject_RDS-SG`
        - Additional configuration -> Initial database name: `wordpress`
> Connection endpoint: `jeremproject-wordpress.caguqn4utdyr.eu-west-1.rds.amazonaws.com`


## 3. Create an EC2 Instance:
- AMI: Amazon Linux 2023 AMI
- Architecture: 64-bit
- Instance type: `t3.small`
- Key pair: `Jerem_CC`
- Security Group: `JeremProject_EC2-SG`


## 4. Create EFS 'JeremProject_EFS'
- Lifecycle management: 'None / None / None'
- Throughput mode: 'Elastic'
- Access points: change the Security Group to `JeremProject_EFS-SG`
> DNS name: `fs-0d22b6f9cb34eb030.efs.eu-west-1.amazonaws.com`


## 5. Attach the EFS to EC2:
- From the EC2 Instance terminal:<br>
`sudo mkdir -p /var/www/html`
- Mount the file system with the following command:

```
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <file_system_id>.efs.<region>.amazonaws.com:/ /var/www/html/
```

In my scenario: 
```
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0d22b6f9cb34eb030.efs.eu-west-1.amazonaws.com:/ /var/www/html/
```

- Check if the EFS is successfully mounted:  
`df -T -h`

![df -T -h](https://i.imgur.com/gMuog5j.png)

It works!

### 5.1. Auto-mount the EFS whenever the EC2 instance gets rebooted
- Install the `amazon-efs-utils` package if not already installed: `sudo yum install -y amazon-efs-utils`
- Edit `/etc/fstab` and add the following line:
```
fs-0d22b6f9cb34eb030.efs.eu-west-1.amazonaws.com:/ /var/www/html/ efs defaults,_netdev 0 0
```

Now, when the instance gets rebooted, it should automatically mount the EFS.
Let's try it:
- `sudo reboot`
- `df -T -h`

We still see the EFS directory correctly mounted. _Success!_

## 6. Install a LAMP stack on the EC2:
```
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel php-gd mariadb105-server
sudo systemctl start httpd && sudo systemctl enable httpd
```

## 7. Install and configure WordPress on the EC2 instance
- From the EC2 instance terminal:
```
cd /home/ec2-user/
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```

```
cd wordpress
cp wp-config-sample.php wp-config.php
```
`sudo nano wp-config.php`

Here, replace the following lines:
```
/** The name of the database for WordPress */
define( 'DB_NAME', 'database_name_here' );

/** Database username */
define( 'DB_USER', 'username_here' );

/** Database password */
define( 'DB_PASSWORD', 'password_here' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );
```
with the appropriate RDS credentials; respectively:
- DB_NAME: `wordpress`
- DB_USER: `admin`
- DB_PASSWORD: `1234567890`
- DB_HOST: the connection endpoint of the RDS DB (in my case: `jeremproject-wordpress.caguqn4utdyr.eu-west-1.rds.amazonaws.com`

Also replace these lines:
```
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );
```
with the randomly generated ones found [here](https://api.wordpress.org/secret-key/1.1/salt/)

Finally, add the following line to the config:
`define('FS_METHOD', 'direct');`

## 8. Deploying WordPress:
- Remove the downloaded archive and copy WordPress application files into the `/var/www/html` directory used by Apache:
```
cd /home/ec2-user
rm latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
rm -rf wordpress
```
- Change user:group of `/var/www/` so that apache(httpd) can access the files: `sudo chown -R apache:apache /var/www/`

- Edit `/etc/httpd/conf/httpd.conf` and replace lines 130 and 156 from `AllowOverride None` to `AllowOverride All`:
```
sudo nano +130 /etc/httpd/conf/httpd.conf
sudo nano +156 /etc/httpd/conf/httpd.conf
```

Then, restart Apache Web Server: `sudo systemctl restart httpd`

Now, when accessing the public DNS address of our EC2 instance, WordPress is successfully installed and we can configure it according to our needs.

- Configure WordPress important settings:

`sudo nano /etc/php.ini`

Then find and replace these lines:
```
upload_max_filesize = 1000M
post_max_size = 1000M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```
Then restart Apache: `sudo systemctl restart httpd`

Success!<br>
![healthCheck](https://i.imgur.com/OBhCgse.png)


## 9. Create Application Load Balancer and register the EC2 instance in the Target Group
- Create Load Balancer (type: **Application Load Balancer**) `JeremProject-ELB`
- Create new Target Group `JeremProject-TG`
    - Target type: Instances
    - Protocol: **HTTP (port 80)**
    - Advanced health check settings:
        - Success codes: `200,301,302`
    - Register the created EC2 Instance target in the Target Group
- Map the Availability Zones (in my context: `eu-west-1a` and `eu-west-1b`)
- Assign the `JeremProject_ALB-SG` to the Load Balancer (and remove the "default" one)
- Setup a listener of port 80 for Now
- Create the Load Balancer
<br>

- Once it is active, we should be able to access our WordPress website by accessing the ALB domain name in our browser.
- Let's access the `/wp-admin` page, sign in and go to Settings > General
- Here, replace WordPress Address and Site Address URL with the DNS name of the ALB -> Save Changes.

## 10. Register a Domain Name in Route53:
- Nico did that for me, so I won't documentate the process here, even though it's quite straightforward; you basically just need to purchase the domain name you want (as long as it's available).
> The domain name is `proitstudents24.net`.

### 10.1. Create a Record Set in Route53:
- Go to your domain name, then hit "Create records":
    - Record name: `www`
        - Route traffic to: "Alias to Application Load Balancer"
            - Choose the appropriate region (here: `eu-west1`)
        - Choose your previously created ALB which should pop up in the dropdown menu.


## 11. Update your WordPress website to use this domain name: 
- Go back to your WP website (/wp-admin) by accessing it from the domain name created.
- "General > Settings" again, then replace WordPress Address and Site Address by the domain name (here: `http://www.proitstudents24.net`) 
- When logging in again, we see that our website now successfully uses our Route53 domain's address. Yay!


## 12. Request an SSL certificate from ACM (AWS Certificate Manager):
- Go to ACM and "Request a public certificate"
- Enter your domain name and hit "Add another domain name to this certificate"
- In this second box, we'll be adding a wildcard `*` before our domain name.
- Then hit "Request". It should say "Pending validation".
- Next, hit "Create records in Route 53"
- Select your domain name and hit "Create record".
- The Status of the Certificated should be now "Issued", and show "Success" for both our records. Success!


## 13. Create an HTTPS Listener for the Application Load Balancer:
- From the EC2 Dashboard > Load Balancer, choose your ALB and click "Add listener".
- Switch the protocol to HTTPS (port 443)
- Under "Default actions", select "Forward" and select your Target Group (`JeremProject_TG1`).
- Under "Default SSL/TLS certificate", choose "From ACM" then select the certificate previously created with ACM and then click "Add".

### 13.1. Update the HTTP listener to redirect HTTP traffic to HTTPS:
- Select your HTTP listener and click "Edit"
- Remove the Default actions
- Select "Redirect" under "Default Actions"
- Make it redirect to HTTPS on port 443.
- Save changes.

## 14. Modify your `wp-config.php` file to make sure your website is secured:
- SSH into your EC2 instance, then type `sudo nano /var/www/html/wp-config.php`
- Under the following lines: 
```php
* @package WordPress
*/
```
add the following lines: 
```php
/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if(isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}`
```
- Save the updated file. 
- Restart the apache webserver: `sudo systemctl restart httpd`.
- Now, when you access your website through its domain name (www.proitstudents24.net), you should be that it's successfully secured. 

## 14.1. Modify your WordPress Address and Site Address:
- Access your WordPress website through its domain name
- Settings > General
- Replace the WordPress Address and Site Address by the ones with `https://`, namely:
    - WP Address: https://www.proitstudents24.net
    - Site Address: https://www.proitstudents24.net
- Save changes.
- Now, you should be able to successfully login to your WordPress website via its HTTPS address. Great!

## 15. Create an AMI of your EC2 instance:
- Once the website is correctly configured, create an image (AMI) of that instance:
- Image Name: `JeremProject_WP_AMI` (Jerem_WP)
- Check `No reboot: Enable`
- Create AMI 

## 16. Create a Launch Template with this AMI:
- From the EC2 dashboard, go to "Launch Templates"
- Hit "Create Launch Template"
- Set a name and description (`JeremProject_LaunchTemplate`)
- Tick _"Provide guidance to help me set up a template that I can use with EC2 Auto Scaling"_
- Choose the AMI created above
- Choose `t3.small` in Instance Type to make sure all the instances created in the future are of the same type
- Choose your key pair (`Jerem_CC`) to include in the Launch Template
- Don't include any suibnet in the Launch Template
- Select the appropriate Security Group (`JeremProject_EC2-SG`) to include
- Hit "Create Launch Template"

## 17. Create an AutoScaling group:
- From the EC2 Dashboard, head to "Auto Scaling Groups"
- "Create Auto Scaling group"
- Give it a name (`JeremProject_ASG`), and choose the previously created Launch Template (`JeremProject_LaunchTemplate`).
- Select your desired availability zones (here: `eu-west-1a` and `eu-west-1b`).
- Load Balancing: select "Attach to an existing Load Balancer"
- There, choose your created Target Group (`JeremProject-TG`)
- Tick `Turn on Elastic Load Balancing health checks`
- Set your Group size as required; my configuration is the following:
    - Desired capacity: 2
    - Minimum capacity: 1
    - Maximum capacity: 3
- Add notifications (if you want to): 
    - I already had an SNS topic where my private email is subscribed to notifications (`JeremProject_SNSTopic`), so I chose this one and ticked all the 'Event Types' to be informed in real time whenever something happens in my AutoScaling group.
- Hit "Create Auto Scaling group".
- Now, you should have two instances running, on the two different availability zones you chose. 
> You can now safely stop your "initial" EC2 instance, as it's not required anymore.

#### Great, our infrastructure is now up and running!
Now, feel free to configure your WordPress website as you like. 

Personally, I installed a theme, a few plugins (including WooCommerce) and setup my basic Pokémon ecommerce shop.

I also added the "WP Offload Media Lite" plugin, that will be helpful for our next steps.

## 18. Implementing Amazon Rekognition via a Lambda function:
The next step is a bit more complex and involves a bit of setup. Let's describe how it's done:
- First, I install the [WP Offload Media Lite](https://wordpress.org/plugins/amazon-s3-and-cloudfront/) plugin on my WordPress website, and configured it appropriately. To do that:
    - I created an **Access Key** for my user (`user04`) on the AWS IAM dashboard.<br>
It is made of an **Access Key ID** and a **Secret Key**, which I configured in my WordPress database (via the plugin's configuration page) - I know it isn't the most secure approach, but that's the one I chose for convenience.
    - Then, I configured the plugin to offload the future media files uploaded to my WordPress website to an S3 bucket (`htwws2324`, as when trying to configure my "own" bucket, I faced some "Not Authorized" errors).
<br>

- After that, I created a Lambda function that I configured with an S3 bucket trigger (the `htwws2323` one), that basically uses the Amazon Rekognition SDK (`boto3`) to analyze the labels of the pictures in the S3 bucket directory that I defined in my Lambda function (`JeremProject_Rekognition`), initializes the Rekogniction client (via the `boto3` SDK), lists all the objects in the specified S3 directory, filters the files by the only extensions that Rekognition can analyze (`.png`, `.jpg` and `.jpeg`), then iterates over each object in the directory, extracts the file name (what's after the last `/`, ignoring the whole path to the image), checks if the image is with an allowed extention AND doesn't contain a hyphen `-` in its name (as with my WordPress configuration, it usually duplicates the images multiple times for "smushing" them in different resolutions for reusability among the site), then calls Rekognition to describe the images labels (`rekognition.detect_labels()`), extracts the labels from the respons, formats the message (using f-strings) for logging and SNS readability, then finally publish that message to my configured SNS topic (`JeremProject_SNSTopic`) by using the `publish_to_sns` function.

Here's how my Lambda function looks like:
```py
import os
import boto3
import json

def publish_to_sns(message, sns_topic_arn):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=sns_topic_arn,
        Message=message,
    )

def lambda_handler(event, context):
    # Specify the directory path and S3 bucket name
    directory_path = 'wp-content/uploads/'
    bucket_name = 'htwws2324'
    sns_topic_arn = 'arn:aws:sns:eu-west-1:609980354649:JeremProject_SNSTopic'  # Replace with your SNS topic ARN

    # Initialize Rekognition client
    rekognition = boto3.client('rekognition')

    # List all objects in the specified S3 directory
    s3 = boto3.client('s3')
    objects = s3.list_objects_v2(Bucket=bucket_name, Prefix=directory_path)

    # Allowed image file extensions
    allowed_extensions = ['.png', '.jpg', '.jpeg']

    # Iterate over each object in the directory
    for obj in objects.get('Contents', []):
        # Extract the filename (what's after the last '/')
        filename = os.path.basename(obj['Key'])

        # Check if the object is an image file with allowed extension and without a hyphen in the filename
        if any(obj['Key'].lower().endswith(ext) for ext in allowed_extensions) and '-' not in filename:
            # Call Rekognition to describe the image
            response = rekognition.detect_labels(
                Image={
                    'S3Object': {
                        'Bucket': bucket_name,
                        'Name': obj['Key']
                    }
                }
            )

            # Extract labels from the response
            labels = [label['Name'] for label in response['Labels']]

            # Format the message for SNS
            message = f"Labels for image {filename}: {labels}"

            # Prints the message for debugging:
            print(message)

            # Publish the message to the configured SNS topic
            publish_to_sns(message, sns_topic_arn)

    print("Processing completed.")
```

Here's what the output looks like on my personal email when I upload a picture of a Lamborghini Revuelto to my WordPress website:
![Lambda_SNS_Output](https://i.imgur.com/9sFqtIz.png)

We can see that it successfully analyzes the labels of the image with a quite good accuracy.

I would have loved to implement a way to add these labels as alternative text for my uploaded WordPress pictures automatically, but didn't have the time to wrap my head around the way the WordPress database structure is like.

## 19. Implementing Infrastructure-as-Code:
To implement IaC (Infrastructure-as-Code), I decided to use **CloudFormation** to create a stack where I created a template to deployed additional pre-configured EC2 instances on-the-go whenever I need to, with quite a lot of flexibility as I can reuse and update the template to my needs if I need to change anything. I won't go into the details, but here's basically what my template is looking like (it basically automatizes all the steps 3, 5, 5.1, 6, 7 and 8 of this documentation); I decided to use `JSON` rather than `YAML` as I'm more familiar with this syntax format:
```json
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Parameters": {
    "KeyName": {
      "Type": "AWS::EC2::KeyPair::KeyName",
      "Default": "Jerem_CC"
    }
  },
  "Resources": {
    "MyEC2Instance": {
      "Type": "AWS::EC2::Instance",
      "Properties": {
        "ImageId": "ami-0c24ee2a1e3b9df45",
        "InstanceType": "t3.small",
        "KeyName": { "Ref": "KeyName" },
        "SecurityGroupIds": ["JeremProject_SG1"],
        "UserData": {
          "Fn::Base64": {
            "Fn::Join": [
              "",
              [
                "#!/bin/bash\n",
                "sudo mkdir -p /var/www/html\n",
                "sudo yum install -y amazon-efs-utils\n",
                "sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0d22b6f9cb34eb030.efs.eu-west-1.amazonaws.com:/ /var/www/html/\n",
                "echo \"fs-0d22b6f9cb34eb030.efs.eu-west-1.amazonaws.com:/ /var/www/html/ efs defaults,_netdev 0 0\" | sudo tee -a /etc/fstab\n",
                "sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel php-gd mariadb105-server\n",
                "sudo systemctl start httpd && sudo systemctl enable httpd\n",
                "cd /home/ec2-user/\n",
                "wget https://wordpress.org/latest.tar.gz\n",
                "tar -xzf latest.tar.gz\n",
                "cd wordpress\n",
                "cp wp-config-sample.php wp-config.php\n",
                "sed -i 's/database_name_here/wordpress/' wp-config.php\n",
                "sed -i 's/username_here/admin/' wp-config.php\n",
                "sed -i 's/password_here/1234567890/' wp-config.php\n",
                "sed -i 's/localhost/jeremproject-wordpress.caguqn4utdyr.eu-west-1.rds.amazonaws.com/' wp-config.php\n",
                "curl -sS https://api.wordpress.org/secret-key/1.1/salt/ | sudo tee -a wp-config.php\n",
                "echo \"define('FS_METHOD', 'direct');\" | sudo tee -a wp-config.php\n",
                "cd /home/ec2-user\n",
                "rm latest.tar.gz\n",
                "sudo cp -r wordpress/* /var/www/html/\n",
                "rm -rf wordpress\n",
                "sudo chown -R apache:apache /var/www/\n",
                "sudo sed -i 's/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf\n",
                "sudo sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 1000M/' /etc/php.ini\n",
                "sudo sed -i 's/post_max_size = 8M/post_max_size = 1000M/' /etc/php.ini\n",
                "sudo sed -i 's/memory_limit = 128M/memory_limit = 256M/' /etc/php.ini\n",
                "sudo sed -i 's/max_execution_time = 30/max_execution_time = 300/' /etc/php.ini\n",
                "sudo sed -i 's/max_input_time = 60/max_input_time = 300/' /etc/php.ini\n",
                "sudo systemctl restart httpd\n"
              ]
            ]
          }
        }
      }
    }
  }
}
```

When created, this stack automatically deploys a preconfigured WordPress EC2 instance with the following configuration:
- Uses my `Jerem_CC` key file
- Uses the Amazon Linux 2023 AMI
- Uses an Instance Type of `t3.small` 
- Attaches this instance to my `JeremProject_SG1` Security Group (very unrestrictive one, so that I can check if it works on-the-go)

Then, it executes all the commands I had to execute to configure that EC2 instance to my needs (mount the EFS, install a LAMP stack, install WordPress, configures the WP Database to use my RDS, edits my PHP settings according to my needs, then restarts the apache web server so the instance deployed is already fully operational and ready to be used.)

---

### References:
- https://aws.amazon.com/tutorials/deploy-wordpress-with-amazon-rds
- https://medium.com/analytics-vidhya/wordpress-on-aws-with-high-availability-35ec5732f1f6
- https://docs.aws.amazon.com/linux/al2023/ug/ec2-lamp-amazon-linux-2023.html
- https://docs.aws.amazon.com/linux/al2023/ug/SSL-on-amazon-linux-2023.html
- https://www.clickittech.com/aws/create-scalable-hosting-wordpress-aws/
- https://www.martinchung.com/2020/10/load-balanced-ssl-wordpress-on-aws/
- https://blog.lawrencemcdaniel.com/wordpress-aws-elb-ssl/
- https://medium.com/@e-miguel/deploy-a-wordpress-website-on-aws-ec7859814410
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html
- https://aws.amazon.com/rekognition/resources/
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html
