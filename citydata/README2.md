including EC2 instance, volumes and security group:

```bash
ansible-playbook stack.yml -i ENV --ask-vault-pass
```
where ENV = dev, test or prod. E.g. test

When prompted, enter the Ansible vault password for your project.

If successful this step will print the private IP address, and if available the public IP address, of the CityData EC2 instance created.

Copy the private IP address into the relevant inventory and group_vars files for the environment you are creating (dev, test or prod).

If available, use the public IP address for connecting with PuTTY (below), otherwise use the private IP address to connect through a VPN.

## Prepare server

### Login to the new server (optional)

Use PuTTY or similar to SSH from your laptop into the new CityData server. You will need a local copy of the key file CityData.ppk (or CityData.pem for Macs).

For PuTTY use the following settings:
* Session > Host name: *CityData's private IP* (or public IP if no VPN is used)
* Session > Connection type: SSH
* Connection > Seconds between keepalives: 120
* Connection > Data > Auto-login username: ubuntu
* Connection > SSH > Auth > Private key file for authentication: *path/to/CityData.ppk*

The first time you SSH to the new server you will be asked to confirm.

### Authorise the control machine to SSH to the server

Ensure the Ansible control machine has the CityData public key:
```bash
ls ~/.ssh/CityData_nonProd.pem
```

Test the connection. On the Ansible control machine:
```bash
ssh <CityData private IP>
```

You will see a warning that the authenticity of the host can't be established.

Type `yes` when prompted to permanently add the IP address of the CityData server to the list of known hosts.

You should now be logged into the CityData server.

Type `exit` to return to the Ansible control machine.

```bash
ansible-playbook prep.yml -i ENV
```

## Using a Python virtual environment

To setup your project using a local python virtual environment, follow these instructions:

1. Prepare the Environment

```
    . .bashrc
    git clone https://github.com/GeoNode/geonode-project.git -b master
    mkvirtualenv citydata
    pip install Django==1.11.16

    django-admin startproject --template=./geonode-project -e py,rst,json,yml,ini,env,sample -n Dockerfile citydata

    cd citydata
```

2. Setup the Python Dependencies

**Don't use sudo** with these commands.

```
pip install -r requirements.txt --upgrade
pip install -e . --upgrade

sudo apt-get install -y libgdal-dev

GDAL_VERSION=`gdal-config --version`
PYGDAL_VERSION="$(pip install pygdal==$GDAL_VERSION 2>&1 | grep -oP '(?<=: )(.*)(?=\))' | grep -oh $GDAL_VERSION\.[0-9])"
pip install pygdal==$PYGDAL_VERSION
```

# Using Custom Local Settings
```
cp citydata/local_settings.py.sample citydata/local_settings.py

vim citydata/local_settings.py
--> add private IP to ALLOWED_SITES
```
Apply patch https://github.com/GeoNode/geonode/pull/4154/commits/61c38088cae628aa165b2b1bbb1d73bcff27298e#diff-1aeb9692c529ee0f5d825a51cb992105 then

```
touch citydata/wsgi.py

DJANGO_SETTINGS_MODULE=citydata.settings paver reset
DJANGO_SETTINGS_MODULE=citydata.settings paver setup
DJANGO_SETTINGS_MODULE=citydata.settings paver sync --> password authentication failed for user "geonode"
DJANGO_SETTINGS_MODULE=citydata.settings paver start --> password authentication failed for user "geonode"
```

3. Access GeoNode from browser:

```
    http://localhost:8000/
    ```

**Note: default admin user is ``admin`` (with pw: ``admin``)**

## Start your server

You need Docker 1.12 or higher, get the latest stable official release for your platform.

1. Prepare the Environment

```

    git clone https://github.com/GeoNode/geonode-project.git -b master
    mkvirtualenv citydata
    pip install Django==1.11.16

    django-admin startproject --template=./geonode-project -e py,rst,json,yml,ini,env,sample -n Dockerfile citydata

    cd citydata
```

2. Run `docker-compose` to start it up (get a cup of coffee or tea while you wait)

   Remember to update "wsgi.py" in case you are using "local_settings"
   vim citydata/wsgi.py
   --> os.environ.setdefault("DJANGO_SETTINGS_MODULE", "citydata.local_settings")

 ```
     docker-compose build --no-cache
     docker-compose up -d

   .. code-block: none

      set COMPOSE_CONVERT_WINDOWS_PATHS=1
```
   before running docker-compose up

3. Access the site on http://localhost/


If you want to run the instance on a public site
------------------------------------------------

Preparation of the image (First time only)
------------------------------------------

.. note: In this example we are going to publish to the public IP http://123.456.789.111

```

  vim docker-compose.override.yml
    --> replace localhost with 123.456.789.111 everywhere
```
## Startup the image

```

  docker-compose up --build -d

  ```

## To Stop the Docker Images

```

  docker-compose stop

  ```

## To Fully Wipe-out the Docker Images

.. warning: This will wipe out all the repositories created until now.

.. note: The images must be stopped first

```

  docker system prune -a

```

## Recommended: Track your changes

Step 1. Install Git (for Linux, Mac or Windows).

Step 2. Init git locally and do the first commit:
```
    git init

    git add *

    git commit -m "Initial Commit"
```
Step 3. Set up a free account on github or bitbucket and make a copy of the repo there.

## Hints: Configuring Requirements.txt

You may want to configure your requirements.txt, if you are using additional or custom versions of python packages.  For example:
```
    Django==1.11.16
    six==1.10.0
    django-cuser==2017.3.16
    django-model-utils==3.1.1
    pyshp==1.2.12
    celery==4.1.0
    Shapely>=1.5.13,<1.6.dev0
    proj==0.1.0
    pyproj==1.9.5.1
    pygdal==2.2.1.3
    inflection==0.3.1
    git+git://github.com/<your organization>/geonode.git@<your branch>
```

## Hints: Using Ansible

You will need to use Ansible Role in order to run the playbook.

In order to install and setup Ansible, run the following commands:
```
    sudo apt-get install software-properties-common
    sudo apt-add-repository ppa:ansible/ansible
    sudo apt-get update
    sudo apt-get install ansible
```
A sample Ansible Role can be found at https://github.com/GeoNode/ansible-geonode

To install the default one, run:
```
    sudo ansible-galaxy install GeoNode.geonode
```
you will find the Ansible files into the ``~/.ansible/roles`` folder. Those must be updated in order to match the GeoNode and GeoServer versions you will need to install.

To run the Ansible playbook use something like this:
```
    ANSIBLE_ROLES_PATH=~.ansible/roles ansible-playbook -e "gs_root_password=<new gs root password>" -e "gs_admin_password=<new gs admin password>" -e "dj_superuser_password=<new django admin password>" -i inventory --limit all playbook.yml
```