# Deploying Multi-OS applications to Docker EE
Docker EE 17.06 is the first Containers-as-a-Service platform to offer production-level support for the integrated management and security of Linux AND Windows Server Containers.

In this lab we'll build a Docker EE cluster comprised of Windows and Linux nodes. 

> **Difficulty**: Intermediate (assumes basic familiarity with Docker)

> **Time**: Approximately 30 minutes

> **Tasks**:
>
> * [Prerequisites](#prerequisites)
> * [Task 1: Build a Docker EE Cluster](#task1)
>   * [Task 1.1: Install the UCP manager](#task1.1)
>   * [Task 1.2: Install a Linux worker node](#task1.2)
>   * [Task 1.3: Install a Windows worker node](#task1.3)
>   * [Task 1.4: Install DTR and Create Two Repositories](#task1.4)
>   * [Task 1.5: Install Self Signed Certs on All Nodes](#task1.5)


## Document conventions

- When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value.

	For instance if you see `<linux vm dns name>` you would actually type something like `pdx-lin-01.uswest.cloudapp.azure.com`

- When you see the Linux penguin all the following instructions should be completed in one of your Linux VMs

	![](./images/linux75.png)

- When you see the Windows flag all the subsequent instructions should be completed in one of your Windows VMs.

	![](./images/windows75.png)


### Virtual Machine Naming Conventions
Your VMs are named in the following convention prefix-os-cluster number-node id.westus2.cloudapp.azure.com

* **Prefix** is a unique prefix for this workshop
* **OS** is either lin for Linux or win for Windows
* **Cluster** number is a two digit number unique to your VMs in this workshop
* **Node ID** is a letter that identifies that node in your cluster

When this guide refers to `<linux node b>` that would be the node with the OS code `lin` and the node ID of `b` (for example `pdx-lin-01-b`>

### Virtual Machine Roles
This lab uses a total of three virtual machines

The Docker EE cluster you will be building will be comprised of three nodes - a Linux manager, a Linux Worker and a Windows worker.

* The **B** nodes are your worker nodes
* The **C** node is you manager node

![](./images/vm_roles.png)

## <a name="prerequisites"></a>Prerequisites

You will be provided a set of three virtual machines (one Windows and two Linux), which are already configured with Docker and some base images. You do not need Docker running on your laptop, but you will need a Remote Desktop client to connect to the Windows VM, and an SSH client to connect into the Linux one.
### 1. RDP Client

- Windows - use the built-in Remote Desktop Connection app.
- Mac - install [Microsoft Remote Desktop](https://itunes.apple.com/us/app/microsoft-remote-desktop/id715768417?mt=12) from the app store.
- Linux - install [Remmina](http://www.remmina.org/wp/), or any RDP client you prefer.

### 2. SSH Client

- Windows - [Download Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html)
- Linux - Use the built in SSH client
- Mac - Use the built in SSH client

> **Note**: When you connect to the Windows VM, if you are prompted to run Windows Update, you should cancel out. The labs have been tested with the existing VM state and any changes may cause problems.

## <a name="task1"></a>Task 1: Build a Docker EE Cluster

In this first step we're going to install Docker Universal Control Plane (UCP) and Docker Trusted Registry. UCP is a web-based control plane for Docker containers that can deploy and manage Docker-based applications across Windows and Linux nodes. Docker Trusted Registry is a private registry server for story your Docker images.

We'll start by installing the UCP manager. Next we'll add a Linux worker node, followed by installing Docker Trusted Registry. Then we'll add a Windows worker node, and finally we'll add our self-signed certs to each of the nodes to ensure they can communicate securely with DTR.

> **Note**: In the current version of UCP manager nodes must be Linux. Worker nodes can be Windows or Linux

### <a name="task1.1"></a>Task 1.1: Install the UCP manager

![](./images/linux75.png)

1. Either in a terminal window (Mac or Linux) or using Putty (Windows) SSH into Linux node **C** using the fully qualified domain name (fqdn). The fqdn should have been provided to you. The username is `docker` and the password is `Docker2017`

	`ssh docker@<linux node c fqdn>`

Start the UCP installation on the current node, and make it your UCP manager.

1.	Pull the latest version of UCP by issuing the following command:

	```
	$ docker image pull docker/ucp:2.2.0
	```

	You should see output similar to below

  	```
	2.2.0: Pulling from docker/ucp
	88286f41530e: Pull complete
	5ee9b2a2067a: Pull complete
	8f1603d2f00b: Pull complete
	Digest: sha256:a41a08269b39377787203517c989a0001d965bb457b45879ab3a7c1e244c9599
	Status: Downloaded newer image for docker/ucp:2.2.0
	```

2.	Install the UCP manager by issuing the following command.

	> **Note**: Be sure to substitute the **Private IP Address** of Linux Node **C** for the `--host-address`

	```
	$ docker container run --rm -it --name ucp \
	-v /var/run/docker.sock:/var/run/docker.sock \
	docker/ucp:2.2.0 install \
	--admin-username docker \
	--admin-password Docker2017 \
	--host-address <linux node c private IP address> \
	--interactive
	```

The installer will pull some images, and then ask you to supply additional subject alternative names (SANs)

4. When prompted for `Additional aliases` you need to supply BOTH the **PUBLIC IP** and **Fully Qualified Domain Name (FQDN)** of Linux Node **C** separated by spaces

	> **Note**: THIS 	STEP IS VERY IMPORTANT PLEASE FOLLOW THESE DIRECTIONS CAREFULLY

	As an example:

	```
	You may enter additional aliases (SANs) now or press enter to proceed with the above list.
  	Additional aliases: 52.183.42.41 pdx-lin-01-c.westus2.cloudapp.azure.com
  	```

You'll see some additional output as the UCP manager is installed


	INFO[0000] Initializing a new swarm at 10.0.2.7
	INFO[0005] Establishing mutual Cluster Root CA with Swarm
	INFO[0008] Installing UCP with host address 10.0.2.7 - If this is incorrect, please specify an alternative address with the '--host-address' flag
	INFO[0008] Generating UCP Client Root CA
	INFO[0010] Deploying UCP Service
	INFO[0058] Installation completed on new-lin-01-c (node iiuinrv3osaoon2meboxojh3x)
	INFO[0058] UCP Instance ID: 6jw827dm4h13kd4nmhc7o10n6
	INFO[0058] UCP Server SSL: SHA-256 Fingerprint=59:8C:C8:AB:37:6E:EF:6C:FB:AA:5E:C5:48:00:27:16:E3:0D:EA:B0:2F:F1:A2:4B:8C:F3:0E:1A:5B:AC:83:F4
	INFO[0058] Login to UCP at https://10.0.2.7:443
	INFO[0058] Username: docker
	INFO[0058] Password: (your admin password)

The next thing we need to do is upload your Docker EE license. For this workshop we are supplying a short-term license, you can download your own 30-day trial license from the Docker Store.

1. Download the [license file](https://drive.google.com/file/d/0ByQd4O58ibOEMkM4bE5XVnJPbEU/view?usp=sharing) to your local laptop

1. Navigate to the UCP console by pointing your browser at `https://<linux node c public ip address>`

> **Note**: You need to use `https:` NOT `http`

> **Note**: Because UCP uses self-signed SSL certs, your web browser may warn you that your connection is not secure. You will need to click through that warning. An example from Chrome is shown below.

![](./images/ssl_error.png)

2. Log in to UCP with the username `docker` and the password `Docker2017`

3. Click `Upload License`, navigate to your license location, and double-click the license file

Congratulations, you have installed the UCP manager node.

### <a name="task1.2"></a>Task 1.2: Install a Linux worker node

Now that we have a manager node, we'll add a Linux worker node to our cluster. Worker nodes are the servers that actually run our Docker-based applications.

1. From the main UCP dashboard click `Add a node` from the `Add Nodes` box near the bottom left.

	![](./images/add_node.png)

2. Copy the text from the dark box shown on the `Add Node` screen.

	> **Note** There is an icon in the upper right corner of the box that you can click to copy the text to your clipboard

	![](./images/join_text.png)

3. SSH into Linux node **B**.

	`ssh docker@<linux node b fqdn>`

4. Paste the text from Step 2 at the command prompt, and press enter.

	You should see the message `This node joined a swarm as a worker.` indicating you've successfully joined the node to the cluster.

5. Switch back to the UCP console in your web browser and click the `x` in the upper right corner to close the `Add Node` window

6. You should be taken to the `Nodes` screen will will see 2 nodes listed at the bottom of your screen. Your **C** node is the manager, and the **B** node is your worker.

Congratulations on adding your first worker node.

In the next step we'll install and configure a Windows worker node.

### <a name="task1.3"></a>Task 1.3: Install a Windows worker node

![](./images/windows75.png)

Let's add our 3rd node to the cluster, a Windows Server 2016 worker node. The process is basically exactly the same as it was for Linux

1. From the Nodes screen, click the blue `Add node` button in the middle of the screen on the right hand side.

> **Note**: You may notice that there is a UI component to select `Linux` or `Windows`. The lab VMs already have the Windows components pre installed, so you do NOT need to select `Windows`. Just leave the selecton on `Linux` and move on to step 2

2. Copy the text from the dark box shown on the `Add Node` screen.

	> **Note** There is an icon in the upper right corner of the box that you can click to copy the text to your clipboard

	![](./images/join_text.png)

3. Use RDP to log in to Windows node **B**.

4. From the Start menu open a Powershell window

4. Paste the text from Step 2 at the command prompt, and press enter.

	You should see the message `This node joined a swarm as a worker.` indicating you've successfully joined the node to the cluster.

5. Switch back to your web browser and click the `x` in the upper right corner to close the `Add Node` window

6. You should be taken to the `Nodes` screen will will see 3 nodes listed at the bottom of your screen. Your Linux node **C** is the manager, and the **B** Linux and Windows nodes are your workers.

Congratulations on building your UCP cluster. Next up we'll install and configure DTR.

### <a name="task1.4"></a>Task 1.4: Install DTR and Create Two Repositories

![](./images/linux75.png)

Like UCP, DTR uses a single Docker container to bootstrap the install process. In the first step we'll kick off that container to install DTR, and then we'll create two repositories that we'll use later for our Tweet apps we're going to deploy.

1. Switch back to (or reinitiate) your SSH session in to Linux node **C**

2. Pull the latest version of DTR

	`$ docker pull docker/dtr:2.3.0`

	You should see output similar to this:

	```
	2.3.0: Pulling from docker/dtr
	019300c8a437: Pull complete
	e79b5d45af49: Pull complete
	8a7bd66a7244: Pull complete
	f5a08fbc29be: Pull complete
	b39bbe9561c9: Pull complete
	fc937f026406: Pull complete
	e72d13961188: Pull complete
	863b39710b20: Pull complete
	500585597a2b: Pull complete
	Digest: sha256:a473733de1ebadc45cae78ae44907eeef332dff6d029b1575c992118db94cd15
	Status: Downloaded newer image for docker/dtr:2.3.0
	```

3. Run the bootstrap container to install DTR.

	You will need to supply three inputs

	* **--dtr-external URL**: The FQDN of Linux Node **B** (i.e.pdx-lin-01-b.westus2.cloudapp.azure.com)
	* **--ucp-node**: The hostname of Linux Node **B** (This is the first part of the FQDN. For example: pdx-lin-01-b)
	* **--ucp-url**: The URL of the UCP server (Linux node **C**) including the port in the form of `https://<linux c node fqdn>:443` (i.e. https://pdx-lin-01-c.westus2.cloudapp.azure.com:443)


   ```
	$ docker run -it --rm docker/dtr install \
	--dtr-external-url <linux node b FQDN> \
	--ucp-node <linux node b hostname> \
	--ucp-username docker \
	--ucp-password Docker2017 \
	--ucp-url <linux node c / UCP manager URL with port #> \
	--ucp-insecure-tls
	```

	You will see a lot of output scroll by while DTR installs finishing with:

	```
	<output deleted>
	INFO[0160] Successfully registered dtr with UCP
	INFO[0161] Establishing connection with Rethinkdb
	INFO[0162] Background tag migration started
	INFO[0162] Installation is complete
	INFO[0162] Replica ID is set to: 8ec3809e352e
	INFO[0162] You can use flag '--existing-replica-id 8ec3809e352e' when joining other replicas to your Docker Trusted Registry Cluster
	INFO[0185] finished reading output
	```
5. Point your web browser to `https://<linux b fqdn>` and log in with the username `docker` and the password `Docker2017`.

	> **Note**: You need to use `https:` NOT `http`

	> **Note**: Because UCP uses self-signed SSL certs, your web browser may warn you that your connection is not secure. You will need to click through that warning. An example from Chrome is shown below.

Now that DTR is installed, let's go ahead and create a couple of repositories to hold our tweet app images. Repositories are how images are organized within a DTR server. Each image gets pushed to its own repository. Multiple images can be stored within a repository by supplying a different tag to each version.

2. From the left hand menu click `Repositories`

3. Click the green `New repository` button on the right hand side of the screen. This brings up the new repository dialog

4. Under `REPOSITORY NAME` type `linux_tweet_app` and click `Save`

Let's repeat this process to create a repository for our Windows tweet app.

3. Click the green `New repository` button on the right hand side of the screen. This brings up the new repository dialogue.

4. Under `REPOSITORY NAME` type `windows_tweet_app` and click `Save`

Congratulations you've installed Docker Trusted Registry, and have created two new repositories.

### <a name="task1.5"></a>Task 1.5: Install Self Signed Certs on All Nodes

Docker uses TLS to ensure the identity of the Docker Trusted Registry. In a production environment you would use certs that come from a trusted certificate authority (CA). However, by default when you install UCP and DTR they use self-signed certs. These self-signed certs are not automatically trusted by the Docker engine. In order for them to be trusted, we need to copy down the root CA cert from the DTR server onto each node in the cluster. There is a script on each of your nodes that will do this for you

> **Note**: This step is only necessary in POC environments where trusted 3rd party certs are not used

Perform the following steps on all 3 of your Linux nodes (**A**, **B**, **C**)

![](./images/linux75.png)

1. SSH into the Linux node

2. At the command prompt run the `copy_certs` script passing it the fqdn of your linux **B** node.

	`$ ./copy_certs.sh <linux node b fqdn>`

	You should see the following output


		% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
		100  1988  100  1988    0     0   7216      0 --:--:-- --:--:-- --:--:--  7229
		Updating certificates in /etc/ssl/certs...
		1 added, 0 removed; done.
		Running hooks in /etc/ca-certificates/update.d...
		done.

> **Note**: In some cases you may see some Perl warnings in addition to the above output, these can be safely ignored.

3. Log into the DTR server from the command line to ensure the cert was copied correctly. The username should be `docker` and the password `Docker2017`

	> **Note**: Be sure to substitute the FQDN of your **B** Linux node

	> **Note**: If you see an x509 certificate is from an unknown source error the cert didn't copy correctly, just rerun the above command.

	```
	$ docker login <linux node b fqdn>
	Username: docker
	Password: Docker2017
	```

	You should see a `Login succeeded` message

> **Note**: Remember to repeat these steps on all 3 Linux nodes.

![](./images/windows75.png)

Now we need to do something similar on the two Windows nodes. Perform the following steps on each of the two Windows nodes.

1. RDP into the Windows node

2. From the start menu, open a Powershell window

3. Execute the `copy_certs` script

	`c:\copy_certs.ps1 <linux node b fqdn>`

3. Log into the DTR server from the command line to ensure the cert was copied correctly. The username should be `docker` and the password `Docker2017`

	> **Note**: Be sure to substitute the FQDN of your **B** Linux node

	> **Note**: If you see an x509 certificate is from an unknown source error the cert didn't copy correctly, just rerun the above command.

	```
	$ docker login <linux node b fqdn>
	Username: docker
	Password: Docker2017
	```

	You should see a `Login succeeded` message

