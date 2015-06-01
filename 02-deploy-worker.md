# Deploy an EC2 worker

## Provision the server

1. If you don't already have them, set AWS environment variables on your local machine:

  ```
  export AWS_ACCESS_KEY_ID=XXX
  export AWS_SECRET_ACCESS_KEY=XXX
  ```

1. Install Ansible. (If you're in a shell that has the OSP virtual environment active, open a new tab and install Ansible in the regular, system-default, Python 2 environment. Ansible doesn't support Python 3.)

  ```
  pip install ansible
  ```

1. Go back into a shell that has the OSP virtual environment active and change down into `/deploy`. From there, run:

  ```
  ./ec2/ec2.py --refresh-cache
  ```

  If this prints out a big JSON object that lists all of the current EC2 inventory on your account, then everything is good to go.

1. Open `/deploy/vars/ec2.yml`. This file has variables that control the number of "server" and "worker" instance that will get provisioned by the Ansible playbook, and the instance sizes for the two categories. When running OSP extraction jobs, a "server" is a single, centralized instance that "workers" write data into. For now, let's just start a single "server" instance, which we can use to interact with the corpus. Set these values:

  ```yaml
  ec2_server_instance_type: c4.4xlarge
  ec2_server_count: 1
  ec2_worker_count: 0
  ```

1. Next, set the Ansible "vault" password. This is used to decrypt the global variables file at `/deploy/group_vars/all/global.yml`, which contains secret configuration settings that shouldn't be committed to GitHub as bare text. To set the password, create a file at `~/.osp-pw.txt` that contains just the password string. Email david@dclure.org for the password.

1. To make sure the password is configured correctly, run:

  ```
  ansible-vault edit group_vars/all/global.yml
  ```

1. This should decrpyt the file using the password and open it in a vim buffer. For now, the only important setting here is the `ec2_osp_snapshot` - this tells Ansible which EBS snapshot of the OSP data to attach to newly-created servers. Enter in the most recent snapshot, and save the file.

1. Now, let's start a server. From the `/deploy` directory, run the provisioning playbook:

  ```
  ansible-playbook provision.yml
  ```

  This will start the inventory described in the `ec2.yml` file, make a volume from the OSP snapshot, and attach it.

1. When this is done, refresh the EC2 cache again, which forces Ansible to see the newly-created instnace.

  ```
  ./ec2/ec2.py --refresh-cache
  ```

1. Next, run the configuration playbook, which installs and starts the different services and data stores used by the extraction code (Postgres, Elasticsearch, Redis, Supervisor, etc.):

  ```
  ansible-playbook configure.yml
  ```

1. Then, run the deployment playbook, which installs the OSP code:

  ```
  ansible-playbook deploy.yml
  ```

  The first time this is run, the "Install pip dependencies" step will take a very long time, as much as 30-40 minutes. This is because the extraction code uses a number of pretty heavyweight libraries (numpy, scipy, pgmagick) that have to be compiled from source. In the next step, we'll save a "wheelhouse" that contains pre-built versions of these binaries, which will make future deploys very fast.

1. For now, though, you should be able to SSH into the new instance. Copy the IP address listed in the output from the Ansible playbooks, and log in with:

  ```
  ssh ubuntu@<IP>
  ```

  And, you'll see an `osp` directory, with the extraction code, with the virtual environment automatically activated on login. Also, hit the IP address of the new instance in a browser, and you should see the web interface for the RQ worker.

## Create a dependency wheelhouse

Next, let's save off a wheelhouse to speed up the deploys.

1. Once you've SSH'ed onto the server, install the "wheel" package:

  ```
  pip install wheel
  ```

1. Then, start a tmux session, since the next step will take a while, and we don't want the session to expire:

  ```
  tmux new -s osp
  ```

1. And then pack the binaries into a wheelhouse:

  ```
  pip wheel -r requirements.txt
  ```

1. Once this finishes, tar up the directory:

  ```
  tar -cvzf osp-wheelhouse.tar.gz wheelhouse
  ```

1. SCP this archive down to your local machine (anywhere is fine). Then, set an environment varialbe that points to the archive:

  ```
  export OSP_WHEELHOUSE=/path/to/archive
  ```

  Now, the next time you provision a new OSP server/worker, the Ansible playbook will automatically deploy this wheelhouse, which will make the deployment step run in about 60 seconds.
