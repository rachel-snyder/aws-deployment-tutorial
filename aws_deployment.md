# AWS Deployment Tutorial

## Definitions

| Name              | What it is                                                                 | Why we need it                                                                                                           |
|-------------------|----------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| AWS EC2 Instance  | Elastic Compute. A virtual machine (VM) hosted by AWS. Each EC2 instance represents a specific location on the internet (IP address) | To host our Django API, our frontend server (Nginx), and our database. We are renting one of Amazon's VMs.                 
| Nginx             | Proxy Server that balances HTTP traffic across multiple servers            | To host our React app / Render requests not prefaced by "/api" / Route requests to URL endpoints prefaced by "/api" to Gunicorn (where our Django API is being hosted) |                                   |
| Gunicorn (Green Unicorn) | Python Web Server Gateway Interface (WSGI) HTTP server / Handles incoming HTTP requests and triggers respective APIViews within Views | To allow Django API to handle requests including handling multiple simultaneous requests / Avoid Django server dropping requests that are made at the same time |
| AWS Route 53       | Domain Name Service (DNS) / Alias to represent IP address on the internet  | To translate a domain/host name into an IP address behind the scenes so you can refer to your website by it's name rather than address                                                                       |
| Certbot           | Client that communicates with the Certificate Authority Let's Encrypt to get HTTPS certificates | To verify the domain from Route 53 is attached to the correct IP address / To have a secure connection (HTTPS rather than HTTP) |


## Steps

### Location: Django settings.py file
- Make sure Django Secret Key is in your .env file and use `os.environ.get("SECRET_KEY")` to get it from your settings file. It should not be accessible from any file you are uploading to version control

- Set `DEBUG = False`

- Set your `ALLOWED_HOSTS`
    - If you know what IP addresses you want to allow, you can set that. If not, you can set it equal to `["*"]` for now to allow your api to receive requests from any host

- Set `CORS_ALLOW_ALL_ORIGINS = True` for now. Later you will want to specify only the IP address of your Nginx server but you don't have that info yet

### Location: Amazon Web Services Website (AWS Management Console)
- Create an AWS account
    - You will be asked for some info including a credit card but you will not be charged anything as long as you stay within the free tier

- Click "View all services" at the bottom of the top left tile, search for EC2 and press "Launch a virtual machine"

- Name your server

- Choose what OS you want your VM to run on (Code Platoon recommends choosing Ubuntu)

- Choose the Free tier for the Machine Image 

- Choose your instance type which determines processing power/memory (Code Platoon recommends choosing t2.micro)

- Under the heading "Key pair (login)" we can set up the keys we need to access our VM securely
    - Choose "Create new key pair"
    - Name your key pair
    - Choose RSA (a very common form of encryption)
    - Select .pem for the file format your private key will be stored in
        - THIS FILE MUST BE SECURE/HIDDEN (it grants access to your VM)
        - Make sure to remember where you keep this file because you won't be able to view it again through the AWS site
        - If you're on a Mac and get a popup asking whether you want to save it to a keychain, you can if you want but it's not important in this case 

- In Network settings, choose from where/what kinds of traffic you want to allow to your VM 
    - Choose where you want to allow SSH traffic from. It's okay to choose "Anywhere" to start but later you may want to change it to only be allowed to come from your own IP address
    - Allow HTTPS traffic
    - You may want to also allow HTTP traffic for now until you make sure you have Certbot and your SSL certificate working properly

- Click the big orange "Launch instance" button on the right of the screen

- From the green success banner at the top of this new page, click on the instance id (the underlined string of characters representing your new EC2 instance)

- Now you will see a row of info about your new instance
    - The instance state may take a moment to change from "pending" to "running"


##### Location: Terminal
- Make sure you're in your home directory using `cd ~`

- When you type `ls -a` you will see a folder named .ssh, this is where you want to move your secret key to

- Copy your secret key file into your .ssh folder using `cp`
    - The following is an example of I was within my .ssh directory making this copy if I had originally downloaded the key into my downloads folder: `cp ~/Downloads/file_name.pem .` (note the dot representing the destination is our current directory)

- Now run `ls -l` to see your permission settings for all your ssh keys
    - They will be displayed in this format: `-rw-r--r--@`
        - The first `-` represents that it is a regular file
        - The `@` at the end represents that there is some additional info about the file (not relevant right now)
        - The 9 characters in between can be broken down into three sections 
            - The first three (`rw-`) represents the *user* permissions (in this case they can Read and Write but not eXecute)
            - The next three (`r--`) indicate that the *staff* permissions (in this case they can only Read)
            - The final three (`r--`) represent the permissions for *anyone* (in this case only Read)
        - We want to set the permissons so that the user (you) is allowed to read and write and no one else can do anything
            - To do this we can use the command `chmod 600 file-name.pem`
                - The `600` comes from the fact that under the hood `rw-r--r--` is just the binary string `110-100-100`. We want our new permissions to represent the binary string `110-000-000` so we convert that string to a 3 digit number with each digit representing each set of 3 characters. We can clearly see that the second two digits are 0 and 0 and the first can be calculated by converting the binary string to a decimal number like we learned last week (in this case 2^1 + 2^2 equals 6) and we get our first digit
            - If instead you want your user to only be allowed to read, you can replace the 600 with 400

- To make your ssh request run the command `ssh -i path_to_file_name.pem remote_user_name@public_dns`
    - You can also find this command on the AWS site if you click on the "Instance ID" section of your instance, then the "Connect" button at the top, it's at the bottom of the "SSH client" tab

- The first time you run this command you may be prompted to confirm whether you want to continue with this unknown host and you should say "yes"

- Now you are within your EC2 instance and you have to install your project and a bunch of tools in here like we did on our own machines during Installfest. If you get a pink popup during any of these just click ok
    - Make sure all of your changes to settings.py have been pushed to GitHub and clone your project into your new VM: `git clone repo_url`
        - If your repo is public you should have no problem, if it's private, either set it to public or go through the process of adding a new SSH key to your GitHub account 
    - We are using the package manager called apt (advanced package tool) since we are now working in linux
    - Update apt: `sudo apt update`
    - Install your Python package manager: `sudo apt install python3-pip`
    - Install PostgreSQL: `sudo apt install postgresql`
    - Create a PostgreSQL superuser with current username: `sudo -u postgres createuser --superuser $USER`
    - Install PostgreSQL dev library: `sudo apt install libpq-dev`
    - Install Nginx's core components: `sudo apt install nginx-core`
    - Install Gunicorn: `sudo apt install gunicorn`
    - Install Python dependencies from requirements file: `pip install -r requirements.txt`
    - Create a PostgreSQL database: `createdb your_db_name`
    - Run your database migration files: `python3 manage.py migrate` (note the use of "python3" rather than the just "python" because we haven't created an alias here)
    - Install Node.js package manager: `sudo apt install npm`
    - Install Node.js version manager: `sudo npm install -g n`
    - Install latest (stable) version of Node.js: `sudo n stable`
    - Install JS dependencies from package.json: `npm install`
    - FINALLY build your project: `npm run build`

- Now we have some files to edit in order to properly configure Nginx
    - For the next few step we're going to use Vim, a popular command line text editor
        - This is the easiest way to edit what we need to right now but if you want to access the files through VS code with a little more work you can

- `cd` into the folder holding your Nginx global config file at `/etc/nginx/`

- Now run the command `sudo vim nginx.conf` to pull up a text editor for the config file and edit the first line to say `user ubuntu` since "ubuntu" is currently the name of the owner of the application files.

- Then above the location section you just edited, add the following lines
    - This creates the functionality of your requests prefaced with "api/" being routed to a different server where the Django API is being hosted
    ```js
    location /api {
        proxy_pass http://0.0.0.0:8000;
    }
    ```

- Quit Vim using `wq` to save your changes

- Run `sudo vim sites-enabled/default`

- Under the location section, edit the line beginning with "try_files" to say `try_files $uri $uri/ /index.html =404;` to specify the the places (in order) Nginx should check for the required files (don't forget the semicolon)

- Now add this line to the top of the location section: `root /usr/share/nginx/html/dist;` (don't forget the semicolon)
    - This line specifies where we want Nginx to look for our project's dist directory but it's currently inaccurate. To correct it, we have to make sure Nginx can find our dist folder where we told it to look and we can do this by copying the folder into that location.
        - To do this `cd` into your home directory using `cd ~/` and run the command in the form: `sudo cp -r path_to_folder/dist/ /usr/share/nginx/html/`

- Run `sudo service nginx restart` to restart Nginx

- Now you should be able to copy your url that looks something like "ec2-3-45-....com" and paste it into your browser to see your website!! You should also be able to use your public IP address to view your site.

- To set up Gunicorn, run the following line: `gunicorn project_name.wsgi --bind 0.0.0.0:8000 --daemon`


### Location: utilities.jsx (OPTIONAL BECAUSE WE WILL DO THIS AGAIN LATER ONCE WE MAKE MORE UPDATES)

- Edit your baseURL so that it reflects the server now hosting our API rather than our local machine. The full line should now look like `baseURL: "http://ip_address:8000/api/" where ip_address is in the form of "ec2-3-45-....com"
    - Push that code to GitHub from your local terminal
    - From your remote server, run `git pull` to sync those changes
    - Then rerun `npm run build` because we changed our front end code
    - Rerun the line in the form:  `sudo cp -r path_to_folder/dist/ /usr/share/nginx/html/` so that  Nginx updates as well
    - Rerun `sudo service nginx restart` to restart the server
    

### Amazon Web Services Website (AWS Management Console)

- Search for Route 53 at the top of the page and click on the service

- Press the orange "Get started" button if you see one

- Select the "Register a domain" option (unless you instead want to transfer an existing domain)

- To register a new domain name, type a domain into the Register Domain section of the Route 53 Dashboard and check to see if it's valid/available

- Once you have found one you want, click "Select" to the right of that name and then the big orange button saying "Proceed to checkout"

- Select the options you want and put in your credit card info
    - Be aware you may have to pay a monthly fee based on how much traffic goes to your domain (you can find more details on that when you review your purchase)
    - If you don't want to buy a domain name there are some free ones you can get online

- You should now see the Route 53 Dashboard and you can click on the button to create a new hosted zone and enter in your new domain name, that you want it to be public, and any other details you want to add

- Now, if you don't see info about this new domain, click on "Hosted zones" on the left tool bar, select your zone, and click view details

- Create a new record for your zone with the value of the IP address of your website. The record type should be "A"

- The "Value" should be the IP address of your website (which can be found in the details of your instance on the AWS site)

- Click the orange "Create records" button

### Location: Certbot Website/Terminal

- On the Certbot Website [https://certbot.eff.org/] you should see a section that says "My HTTP website is running (Software) on (System)"
    - Choose Nginx and Ubuntu 20
    - If you scroll down you will see some instructions
        - In your ec2 instance run `sudo snap install --classic certbot`
        - Run `sudo ln -s ~/snap/bin/certbot ~/usr/bin/certbot`
        - Run `sudo certbot --nginx`
            - Provide your email
            - Agree to the terms
            - Answer the question about sharing your email
            - Provide the domain name you set up with Route 53 as the name you want on your certificate in the form "example.com"

- You should now be able to see "http" be replaced with "https" in your browser

- Run `sudo vim utilities.jsx` to get back into your utilities file and edit the base url to be in the form `"https://example.com/api/"`

- Rerun `npm run build`

- Rerun `sudo cp -r path_to_folder/dist/ /usr/share/nginx/html/

- Rerun `sudo service nginx restart`