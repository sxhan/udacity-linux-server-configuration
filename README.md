# README
Project to fulfill asldjfals;djfs

## TODO:
- Fix postgresql and sqlalchemy interaction resulting in incorrect primary key behavior
- Fix Facebook login error

## IP Address and URL
http://34.198.238.35

## Summary of Software Installed
The following software is installed:
- python
- various python libraries: flask, psycopg2, venv, etc.
- apache2 httpd server
- postgresql

## Project Deployment Playbook
The server was configured following these steps:

1. Update all packages
 - `sudo apt-get update && sudo apt-get upgrade`
2. Change ssh port to 2200 instead of 22
 - `sudo vi /etc/ssh/sshd_config`
 - Change `Port 22` to `Port 2200`
 - Restart: `sudo service sshd restart`
3. Configure lightsail firewall to allow 2200, HTTP (80), and NTP (123)
4. Configure UFW to allow access on the same ports only:
 - Add default policies: `sudo ufw default deny incoming && sudo ufw default allow outgoing`
 - Allow the 3 ports: `sudo ufw allow 2200 && sudo ufw allow http && sudo ufw allow ntp`
 - Activate: `sudo ufw enable`
5. Create grader user and give sudo:
 - Create user: `sudo adduser grader`
  - Follow onscreen prompt. Need to create password but all other prompts can be skipped by hitting enter
 - Give sudo: `sudo usermod -aG sudo grader`
6. Switch to the grader user and create a pair of ssh keys:
 - Create files and dirs `mkdir ~/.ssh && touch ~/.ssh/authorized_keys`
 - Change permissions: `chmod 700 ~/.ssh/ && chmod 644 ~/.ssh/authorized_keys`
 - `cd ~/.ssh`
 - Generate ssh key pair: `ssh-keygen`
  - Feel free to use default save directory and blank password
7. Set timezone to UTC
 - Run: `sudo dpkg-reconfigure tzdata`
  - Select 'None of the above'
  - Follow onscreen prompts to change timezone to UTC
 - After exiting, confirm system date: `date`
8. Backup image in lightsail, since the next steps scan be strange and screw everything up
9. Lets proceed with installing the required packages:
 ```bash
 sudo apt-get -qqy update
 sudo apt-get -qqy install postgresql python-psycopg2
 sudo apt-get -qqy install python-flask python-sqlalchemy
 sudo apt-get -qqy install python-pip
 sudo apt-get -qqy install python-dev
 sudo apt-get -qqy install libffi-dev
 ```
10. Create virtualenv for specific python packages:
 - If not installed, install virtualenv: `sudo pip install virtualenv`
 - Create virtualenv in home directory: `cd ~ && virtualenv venv`
 - Activate virtualenv: `source ~/venv/bin/activate`
11. Install all dependencies:
 ```bash
 pip install bleach
 pip install oauth2client
 pip install requests
 pip install httplib2
 pip install redis
 pip install passlib
 pip install itsdangerous
 pip install flask-httpauth
 pip install flask_login
 pip install flask-seasurf
 pip install bcrypt
 ```
12. Install git: `sudo apt-get install git-all`
13. Create database called catalog. Configure database with a user called catalog with limited access:
 - Create user with password: `sudo su postgres -c 'createuser -DRS catalog -P'`
  - When prompted, enter password
 - `sudo su postgres -c 'createdb catalog'`
14. Install apache and mod_wsgi. Serve a dummy hello world page instead of the default page:
 - Install the applications `sudo apt-get install apache2 && sudo apt-get install libapache2-mod-wsgi`
 - Update the apache2 configuration to serve a mod_wsgi application:
  - `sudo vi /etc/apache2/sites-enabled/000-default.conf`
  - add the following line at the end of the `<VirtualHost *:80>` block, right before the closing `</VirtualHost>` line: `WSGIScriptAlias / /var/www/html/myapp.wsgi`
 - Modify the wsgi app:
   - `sudo vi /var/www/html/myapp.wsgi`
   - Paste in the following:
   ```python
   def application(environ, start_response):
    status = '200 OK'
    output = 'Hello World!'

    response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
    start_response(status, response_headers)

    return [output]
   ```
  - Restart apache2 server: `sudo apache2ctl restart`
14. Checkout catalog-app repository: `cd ~ && git clone git clone https://github.com/sxhan/catalog-app-project.git && cd catalog-app-project`
15. Create local configs directory and add local configs:
 - `mkdir instance`
 - Create a local config file and paste in the following: `vi instance/config.py`:
 ```python
 SECRET_KEY = "not so secret"
 FB_CLIENT_SECRETS = {
     "web": {
         "app_id": "1293669250698238",
         "app_secret": "d56adf9a6e289490202f3edf4ff4e5b0"
     }
     DB_STRING = 'postgresql+psycopg2://catalog:securepassword@localhost/catalog'
 }
 ```
16. Create VirtualServer config for the catalog app:
 - Edit the config file: `sudo vi /etc/apache2/sites-enabled/000-default.conf`
 - Paste in the following
 ```
 <VirtualHost *:80>
         # ServerName mywebsite.com
         WSGIScriptAlias / /var/www/catalog-app-project/wsgi.py
         <Directory  /var/www/catalog-app-project/item_catalog/>
             Order allow,deny
             Allow from all
         </Directory>
         Alias /static /var/www/catalog-app-project/item_catalog/static
         <Directory /var/www/catalog-app-project/item_catalog/static/>
             Order allow,deny
             Allow from all
         </Directory>
         ErrorLog ${APACHE_LOG_DIR}/error.log
         LogLevel warn
         CustomLog ${APACHE_LOG_DIR}/access.log combined
 </VirtualHost>
 ```
17. Copy app into the correct directory: `sudo cp -r ~/catalog-app-project /var/www/.`
18. Change the wsig.py application to look like this:
 ```python
 activate_this = '/home/ubuntu/venv/bin/activate_this.py'
 execfile(activate_this, dict(__file__=activate_this))

 import sys
 print sys.path
 sys.path.insert(0, '/var/www/catalog-app-project/')

 from item_catalog import app as application

 if __name__ == "__main__":
     app.run()
 ```
19. Should be good to go after a restart of the web server: `sudo apache2ctl restart`


## References:
- ufw: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04
- Virtual envs: http://docs.python-guide.org/en/latest/dev/virtualenvs/
- Changing timezone: http://unix.stackexchange.com/questions/110522/timezone-setting-in-linux
- Ubuntu 16.04 server config: https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04
- Flask deployment: http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/ and https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
