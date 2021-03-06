## GROWTH Summer School 2020 Development

### Jupyter-Hub implementation


**IMPORTANT NOTE FOR STUDENTS**: For instruction about the school modules, please see the README file. The documentation below is supposed to serve as support to the development team.

This is what is actually happening when a user logs in and gets a server:

* User logs in at https://growth.dirac.institute/hub/login with GitHub

* If they are authenticated, a docker container is started on a computer (host machine) in a cluster of computers on Amazon Web Services (AWS). The docker image used is (right now) https://hub.docker.com/repository/docker/astronomycommons/growth-hub-notebook. You can imagine a computer on AWS literally running `docker run astronomycommons/growth-hub-notebook` and exposing ports that allow that container (and the Jupyter notebook inside of it) to network with the outside world. The docker container has a filesystem isolated from the host machine it is running on, but we modify the filesystem with mounts to other resources. In this case, the docker container mounts `/home/admin` and `/nfs` to another computer in the cluster that hosts the NFS server. Additionally, `/home/{username}`  (e.g. `/home/stevenstetzler`) is mounted to a hard drive (EBS volume in AWS terms) attached to the host machine. Only these directories are permanent, the rest of the filesystem is ephemeral. We can edit what is contained in the ephemeral file system by making changed to the Docker image itself. For example, to edit the conda installation in `/opt/conda` it’s easiest to add the line `RUN conda install <my-package>` to the [Dockerfile](../scripts/Dockerfile), build, and push the new image. Changes made this way will only propagate to the user after the user restarts their server by navigating to https://growth.dirac.institute/hub/home and clicking "Stop My Server" (since restarting the server triggers the latest image to be pulled from the docker hub). As admins you can also navigate to https://growth.dirac.institute/hub/admin to start/stop/and access other people’s servers.

* When the docker container starts (`docker run` on the host), a chain of scripts get run that customize the user environment before running the jupyter notebook command. In particular, the first command that gets run is `start-singleuser.sh` (https://github.com/jupyter/docker-stacks/blob/master/base-notebook/start-singleuser.sh) which triggers `start.sh` (https://github.com/growth-astro/growth-hub-notebook/blob/master/start.sh) which looks for more scripts in `/usr/local/bin/before-notebook.d`, where we've placed a final script:

		# file: /usr/local/bin/before-notebook.d/start-scripts.sh
		log_dir=/home/admin/logs
		mkdir -p $log_dir
		if [ -f /home/admin/start/as-root.sh ]; then
		    log_file=$log_dir/$NB_USER-as-root.log
		    touch $log_file
		    sudo -E -H -u root /home/admin/start/as-root.sh | tee $log_file
		    sudo -E -H -u root chown admin:admin $log_file
		    sudo -E -H -u root chmod o-rwx $log_file
		fi
		if [ -f /home/admin/start/as-user.sh ]; then
		    log_file=$log_dir/$NB_USER-as-user.log
		    touch $log_file
		    sudo -E -H -u $NB_USER /home/admin/start/as-user.sh | tee $log_file
		    sudo -E -H -u root chown admin:admin $log_file
		    sudo -E -H -u root chmod o-rwx $log_file
		fi

which actually runs the scripts [`/home/admin/start/as-user.sh`](../scripts/as-user.sh) and [`/home/admin/start/as-root.sh`](../scripts/as-root.sh).
To summarize, if you are making permanent changes (adding code), you should modify, build, and push a new Docker image. And if you are making changes to those scripts that need run on server start-up (like pulling notebooks from GitHub or copying the data to the user's home directory), you should make changes to the start-up sequence by editing `as-user.sh` and `as-root.sh`. And to actually test the changes, you should stop and start your notebook server to either trigger pulling the updated Docker image or re-run the start-up scripts at https://growth.dirac.institute/hub/home or https://growth.dirac.institute/hub/admin.

https://github.com/jupyter/docker-stacks/blob/master/base-notebook/start-singleuser.sh <br>
https://github.com/growth-astro/growth-hub-notebook/blob/master/start.sh

to make this process as hands-off as possible, I will set up the system to pull the Docker image `growthastro/growth-school-2020-notebook` with the `deploy` tag so you can independently push and test changes to the base Docker image. And as discussed, the admin account and the network filesystem should let you independently edit those start-up scripts.


### FAQ and examples

* How to update the docker to include new software?
  * You can use `apt-get`. In the [Dockerfile](../scripts/Dockerfile), add:

		RUN apt-get -y install mysoftware
  * Then to update the docker image: <br>
    The Dockerfile represents the “source code” for the Docker  image — this can exist anywhere, and the GitHub repository I set up is just to keep track of the latest version of the Dockerfile
The DockerHub repository is the location to pull a Docker image “binary”
The hub is set up to pull a Docker image from the DockerHub repository growthastro/growth-school-2020-notebook with the tag deploy. In order to change the contents of the image that gets pulled from the Dockerhub repository, one must build a new Docker image from a modified Dockerfile and push the new image to the DockerHub repository. Here are instructions for how to do that:

		# get the Dockerfile
		git clone https://github.com/growth-astro/growth-school-2020.git
		cd growth-school-2020
		cd scripts
		# make edits to the Dockerfile
		# ...
		# build a Docker image from the Dockerfile (in current directory .) and tag it (-t) so Docker knows this Docker image should belong to the 'growthastro/growth-school-2020-notebook' DockerHub repository and should have the tag 'deploy'
		docker build . -t growthastro/growth-school-2020-notebook:deploy
		# push the new version of the Docker image to the DockerHub
		docker push growthastro/growth-school-2020-notebook:deploy

    This will overwrite the remote Docker image stored on DockerHub. The tagging system (deploy) allows different versions of the Docker image to exit on the DockerHub. You can tag images with any name, but for this deployment, the most recently pushed image with tag deploy will be used by the JupyterHub


* How to update python to include new modules?

The easiest way is using `conda` or `pip`. For example, to install `scipy`, edit the Dockerfile as follows. <br>
Before:
		
		RUN conda install --yes \
		    numpy \
		    astropy
		
After:

		RUN conda install --yes \
		    numpy \
		    astropy \
		    scipy

When the change is made, build, and push the new image

* How to update to include basic scripts?

There are several.  I uploaded them to `/nfs/bin` (which is a persistent space) and gave them executable permissions.  Then I added a check to the startup script to make sure that directory is in the path.

* How to update the data file
  * Download the data file from google drive
  * Then upload it to the hub (ideally this could be done in one step with wget/curl, but google's interface often asks questions about virus scans etc that make it harder)
  * expand it in the `/nfs` area for persistance:
  
          (base) dlakaplan@jupyter-dlakaplan:/nfs$ sudo -u admin tar xvf ~dlakaplan/growth-school-2020_Jul23.tar

  * the script [`link_growth_data.py`](../scripts/link_growth_data.py) will copy data to the user's directory
  * Update permissions:
  
          (base) dlakaplan@jupyter-dlakaplan:/nfs/growth-school-2020$ sudo -u admin chmod -R a+r .
	  
* How to add new students: add to the `Participants` at https://github.com/orgs/growth-astro-school/teams
  * The first time an admin logs in we need to grant access to the right organization 
  * Can be controlled through "[Third-party applications](Screen%20Shot%202020-07-24%20at%2011.41.49%20AM.png)" tab in settings for admins
* How to add new Tutors: add to the `Tutors` at https://github.com/orgs/growth-astro-school/teams
* How to (re)start the server to test/deploy changes
  * For an individual look under `Control Panel` (upper right) where you can shut down/restart your server
* How to view the hub as another user (if you have admin rights):
  * Go to `Control Panel` ![Control Panel](control_panel.png)
  * Click on `Admin`
  * You should see everybody's server: ![Control Panel](admin_view.png)
  * You can then enter as somebody else if you need to
* How to give admin rights on the hub:
  * You can change their access at the JupyterHub level by going to https://growth.dirac.institute/hub/admin, clicking “edit user” and enabling the “admin” option. If you want to change their access at the filesystem level, you should add their names (as root) to the `/home/admin/etc/sudoers.d/00-admins-group`.
