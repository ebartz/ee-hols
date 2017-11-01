# Deploying Hybrid-OS Applications with Docker EE

In this lab we'll deploy a variety of different applications with Docker EE. We'll start with a traditional build, ship, run workflow of a Linux application. We'll then use Image2Docker to extract a Windows application from an existing VHD. And, finally, we'll use Docker Compose to deploy an application that includes both Windows and Linux components.

> **Difficulty**: Intermediate (assumes basic familiarity with Docker)

> **Time**: Approximately 30 minutes

> * [Task 1: Configure the Docker EE Cluster](#task1)
>   * [Task 1.1: Accessing PWD](#task1.1)
>   * [Task 1.2: Add a Windows worker node](#task1.2)
>   * [Task 1.3: Create Two Repositories](#task1.3)
> * [Task 2: Deploy a Linux Web App](#task2)
>   * [Task 2.1: Clone the Demo Repo](#task2.1)
>   * [Task 2.2: Build and Push the Linux Web App Image](#task2.2)
>   * [Task 2.3: Deploy the Web App using UCP](#task2.3)
> * [Task 3: Deploy a Windows Web App](#task3)
>   * [Task 3.1: Create the Dockerfile with Image2Docker](#task3.1)
>   * [Task 3.2: Build and Push Your Image to Docker Trusted Registry](#task3.2)
>   * [Task 3.3: Deploy the Windows Web App](#task3.3)
> * [Task 4: Deploy a Multi-OS App](#task4)
>   * [Task 4.1: Examine the Docker Compose File](#task4.1)
>   * [Task 4.2: Deploy the Application Stack](#task4.2)
>   * [Task 4.3: Verify the Running App](#task4.3)

## Understanding the Play With Docker Interface

![]({{site.baseurl}}/images/pwd_screen.png)

There are three main components to the Play With Docker (PWD) interface.

### 1. Console Access
Play with Docker provides access to the three Docker EE hosts in your Cluster. These machines are:

* A Linux-based Docker EE 17.06 Manager node
* A Linux-based Docker EE 17.06 Worker node
* A Windows Server 2016-based Docker EE 17.06 Worker Node

By clicking a name on the left, the console window will be connected to that node.

### 2. Access to your Universal Control Plane (UCP) and Docker Trusted Registry (DTR) servers

Additionally, the PWD screen provides you one-click access to the Universal Control Plane (UCP)
web-based management interface as well as the Docker Trusted Registry (DTR) web-based management interface. Click on either the `UCP` or `DTR` button will bring up the respective server web interface in a new tab.

### 3. Session Information

Throughout the lab you will be asked to provide either hosntnames or login credentials that are unique to your environment. These are displayed for you at the bottom of the screen.

## Document conventions

- When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value.

	For instance if you see `<dtr domain>` you would actually type something like `ip172-18-0-7-b70lttfic4qg008cvm90.direct.microsoft.play-with-docker.com`

- When you see the Linux penguin all the following instructions should be completed in your Linux console.

	![]({{site.baseurl}}/images/linux75.png)

- When you see the Windows flag all the subsequent instructions should be completed in your Windows console.

	![]({{site.baseurl}}/images/windows75.png)

## <a name="task1"></a>Task 1: Configure the Docker EE Cluster

The Play with Docker (PWD) environment is almost completely setup, but before we can begin the labs we need to do two more steps. First, we'll add a Windows node to the cluster, and then we'll create two repositories on the DTR server.

### <a name="task 1.1"></a>Task 1.1: Accessing PWD

1. Open the PWD environment **in a new tab or window** by right-clicking [the PWD environment sign-in page](https://goto.docker.com/2017PWDonMicrosoftAzure_MTALP.html).

	> **Note**: You might want to right click the above link and open it in a new tab or window.

2. Fill out the form, and click `submit`. You will then be redirected to the PWD environment.

3. Click `Access`

	It will take a few minutes to provision out your PWD environment. After this step completes, you'll be ready to move on to step 1.2: Install a Windows worker node.

### <a name="task1.2"></a>Task 1.2: Add a Windows worker node

Let's start by adding our 3rd node to the UCP cluster, a Windows Server 2016 worker node. This node is already built, but not yet added to the cluster. In this section we'll obtain the command required to join it to the cluster, and we'll execute that command.

1. From the main PWD screen click the `UCP` button on the left side of the screen.

	> **Note**: Because this is a lab-based installation of Docker EE, we are using the default self-signed certs. Because of this your browser may display a security warning. It is safe to click through this warning.
	>
	> In a production environment you would use certs from a trusted certificate authority and would not see this screen.
	>
	> ![]({{site.baseurl}}/images/ssl_error.png)

2. When prompted, enter your username and password (these can be found below the console window in the main PWD screen). The UCP web interface should load up in your web browser.

	> **Note**: Once the main UCP screen loads you'll notice there is a red warning bar displayed at the top of the UCP screen. This is an artefact of running in a lab environment. A UCP server configured for a production environment would not display this warning
	>
	> ![]({{site.baseurl}}/images/red_warning.png)

3. From the UCP Dashboard click `Add a Node` on the bottom left of the screen.

	![]({{site.baseurl}}/images/add_a_node.png)

4. Copy the text from the dark box shown on the `Add Node` screen.

	> **Note** There is an icon in the upper right corner of the box that you can click to copy the text to your clipboard.
	>
	> ![]({{site.baseurl}}/images/join_text.png)

	> **Note**: You may notice that there is a UI component to select `Linux` or `Windows` on the `Add Node` screen. In a production environment where you are starting from scratch there are a few prerequisite steps to adding a Windows node. However, we've already done these steps in the PWD environment. So for this lab, just leave the selection on `Linux` and move on to step 2

  ![]({{site.baseurl}}/images/windows75.png)

5. Switch back to the PWD interface, and click the name of your Windows node. This will connect PWD to the terminal running on your Windows Server 2016 Docker EE host.

6. Paste the text from Step 4 at the command prompt in the Windows console.

	You should see the message `This node joined a swarm as a worker.` indicating you've successfully joined the node to the cluster.

7. Switch back to the UCP server in your web browser and click the `x` in the upper right corner to close the `Add Node` window.

8. You should be taken to the `Nodes` screen and will see 3 nodes listed at the bottom of the screen.

	After a minute or two, refresh your web browser to ensure that your Windows worker node has come up as `healthy`.

	![]({{site.baseurl}}/images/node_listing.png)

Congratulations on adding a Windows node to your UCP cluster.

Next up, we'll create a couple of repositories in Docker Trusted registry.

### <a name="task1.3"></a>Task 1.3: Create Two DTR Repositories

Docker Trusted Registry (DTR) is an enterprise-grade registry designed to store and manage your Docker images. In this lab we're going to create a couple of Docker images and push them to DTR. But before we can do that, we need to setup repositories in which those images will reside.

1. In the PWD web interface click the `DTR` button on the left side of the screen.

	> **Note**: As with UCP before, DTR is using self-signed certs. It's safe to click through any browser warning you might encounter.

2. From the main DTR page click `New Repository`. This brings up the new repository dialog.

	![]({{site.baseurl}}/images/create_repository.png)

3. Under `REPOSITORY NAME`, type `linux_tweet_app`. Leave the rest of the values the same, and click `Save`.

Let's repeat this process to create a repository for our Windows tweet app.

1. Once again, click the green `New repository` button.

2. Under `REPOSITORY NAME`, type `windows_tweet_app`. Leave the rest of the values the same, and click `Save`.

  ![]({{site.baseurl}}/images/two_repos.png)

Congratulations you have created two new repositories in your DTR.

## <a name="task2"></a>Task 2: Deploy a Linux Web App

Now that we've configured the cluster, let's deploy a couple of web apps. These are simple web pages that allow you to send a tweet. One is built on Linux using NGINX and the other is build on Windows Server 2016 using IIS.  

Let's start with the Linux version.

### <a name="task2.1"></a> Task 2.1: Clone the Demo Repo

![]({{site.baseurl}}/images/linux75.png)

1. From PWD click on on the `worker1` link on the left to connect your web console to the UCP Linux worker node.

2. Use `git` to clone the workshop repository.

	```
	$ git clone https://github.com/dockersamples/linux_tweet_app.git
	```

	You should see something like this as the output:

	```
	Cloning into 'linux_tweet_app'...
	remote: Counting objects: 13, done.
	remote: Compressing objects: 100% (10/10), done.
	remote: Total 13 (delta 1), reused 10 (delta 1), pack-reused 0
	Unpacking objects: 100% (13/13), done.
	Checking connectivity... done.
	```

	You now have the necessary demo code on your Linux worker host.

### <a name="task2.2"></a> Task 2.2: Build and Push the Linux Web App Image

![]({{site.baseurl}}/images/linux75.png)

1. Change into the `linux_tweet_app` directory.

	```
	$ cd ./linux_tweet_app/
	```

2. Export a new environment variable to make some of the commands that follow more "copy / paste" friendly.

	1. Copy the `DTR Hostname` from the bottom of the main PWD screen.

	2. Use the following command to assign that to the `DTR_HOST` environment variable. Remember to substitute the real value from your environment.

		```
		export DTR_HOST=<dtr hostname>
		```

	3. Create another environment variable, this time containing your username.

		Copy the user name from the bottom of the PWD screen (under credentials it's the first of the two listed).

		```
		export USER=<user name>
		```

	4. Echo both back to ensure they're set.

		```
		echo $DTR_HOST && echo $USER
		```

2. Use `docker image build` to build your Linux Tweet web app Docker image. Be sure to include the period (`.`) at the end of the command.

	```
	$ docker image build -t $DTR_HOST/$USER/linux_tweet_app .
	```

	The `-t` flag tags the image with a name. In our case the name indicates which DTR server and under which user's repository the image will live.

	> **Note**: Feel free to examine the Dockerfile in this directory if you'd like to see how the image is being built.

	Your output should be similar to what is shown below.

	```
	Sending build context to Docker daemon  4.096kB
	Step 1/4 : FROM nginx:latest
	latest: Pulling from library/nginx
	ff3d52d8f55f: Pull complete
	b05436c68d6a: Pull complete
	961dd3f5d836: Pull complete
	Digest: sha256:12d30ce421ad530494d588f87b2328ddc3cae666e77ea1ae5ac3a6661e52cde6
	Status: Downloaded newer image for nginx:latest
	 ---> 3448f27c273f
	Step 2/4 : COPY index.html /usr/share/nginx/html
	 ---> 72d22997a765
	Removing intermediate container e262b9220942
	Step 3/4 : EXPOSE 80 443
	 ---> Running in 54e4ff1b39a6
	 ---> 2b5bd87894cd
	Removing intermediate container 54e4ff1b39a6
	Step 4/4 : CMD nginx -g daemon off;
	 ---> Running in 54020cdec942
	 ---> ed5f550fc339
	Removing intermediate container 54020cdec942
	Successfully built ed5f550fc339
	Successfully tagged  $DTR_HOST/$USER/linux_tweet_app:latest
	```

3. Log into your DTR server from the command line.

	```
	$ docker login $DTR_HOST
	Username: <your username>
	Password: <your password>
	Login Succeeded
	```

4. Use `docker image push` to upload your image up to Docker Trusted Registry.

	```
	$ docker image push $DTR_HOST/$USER/linux_tweet_app
	```

	The output should be similar to the following:

	```
	The push refers to a repository [$DTR_HOST/$USER/linux_tweet_app]
	feecabd76a78: Pushed
	3c749ee6d1f5: Pushed
	af5bd3938f60: Pushed
	29f11c413898: Pushed
	eb78099fbf7f: Pushed
	latest: digest: sha256:9a376fd268d24007dd35bedc709b688f373f4e07af8b44dba5f1f009a7d70067 size: 1363
	```

5. In your web browser, head back to your DTR server and click `View Details` next to your `linux_tweet_app` repo to see the details of the repo.

	> **Note**: If you've closed the tab with your DTR server, just click the `DTR` button from the PWD page.

6. Click on `Images` from the horizontal menu. Notice that your newly pushed image is now on your DTR.

### <a name="task2.3"></a> Task 2.3: Deploy the Web App using UCP

Now let's run our application by creating a new service.

Services are application building blocks (although in many cases an application will only have one service, such as this example). Services are based on a single Docker image. When you create a new service you instantiate at least one container automatically, but you can scale the number up and down to meet the needs of your service.

1. Switch back to your UCP server in your web browser.

	> **Note**: If you've closed your UCP tab, you can simply click `UCP` from the PWD page to re-launch the UCP web interface

2. In the left hand menu click `Services`.

3. In the upper right corner click `Create Service`.

4. Enter `linux_tweet_app` for the name.

4. Under `Image` enter the path to your image which should be `<DTR hostname>/<your username>/linux_tweet_app`.

	> **Note**: You can copy the full image path from the output of your `docker image push` command.

8. From the left hand menu click `Network`.

9. Click `Publish Port+`.

	We need to open a port for our web server. Since port 80 is already used by UCP on one node, and DTR on the other, we'll need to pick an alternate port. We'll go with 8088.

10. Fill out the port fields as shown below.

	![]({{site.baseurl}}/images/linux_ports.png)

11. Click `Confirm`.

12. Click `Create` near the bottom right of the screen.

After a few seconds you should see a green dot next to your service name. Once you see a green dot, you can see the web site by pointing your web browser to port 8088 on the Linux manager node in the cluster: `http://<linux-manager-node>:8088` (it may take a minute or so after the dot turns green for the service to be fully available).

> **Note**: You want to go to `http://` not `https://`

The `linux_tweet_app` is now running as a Docker service.

### Extra Credit: Ingress Load Balancing

In this step we'll see how ingress load balancing allows us to connect to our service via any Linux node in the cluster.

1. In UCP click on `Services` in the left hand menu.

2. From the List of services click on `linux_tweet_app`.

3. From the dropdown on the right-hand side select `Inspect Resources` and then `Containers`. Notice which host the container is running on. Is it running on the manager or the worker node?

	![]({{site.baseurl}}/images/linux_tweet_app_container.png)

	If it's the worker node, how did your web browser find it when we pointed at the UCP Manager node?

4. Point your browser at port 8088 on any of the Linux nodes in the cluster that are not hosting the service. Did the site come up?

	In the end, it doesn't matter if we try and access the service via the manager or the worker, Docker EE will route the request correctly.

	> **Note**: DTR is running on the worker node, so pointing to the DTR server is the same as pointing at the worker node.

	This is an example of the built-in ingress load balancer in Docker EE. Regardless of where a Linux-based service is actually running, you can access it from any Linux node in the cluster. So, if it's running on the manager in our cluster, you can still get to it by accessing the worker node. Docker EE can accept the request coming into any of the Linux nodes in the cluster, and route it to a host that's actually running a container for that service.

5. Be sure to clear the filter in the UCP UI by clicking the `X` in the upper right corner. If you don't do this, you won't see any of the other services you deploy later in the lab.

	![]({{site.baseurl}}/images/clear_filter.png)

## <a name="task3"></a>Task 3: Deploy a Windows Web App

Now we'll deploy the Windows version of the tweet app.

### <a name="task3.1"></a> Task 3.1: Create the Dockerfile with Image2Docker

There is a Windows Server 2016 VHD that contains our Windows Tweet App stored in `c:\` on Windows host. We're going to use Image2Docker to scan the VHD and create a Dockerfile. We'll build an image from the Dockerfile as we did in the previous step, push it to DTR, and then deploy our Windows tweet app.

![]({{site.baseurl}}/images/windows75.png)

1. Click the name of your Windows host in PWD to switch your web console.

2. Use Image2Docker's `ConvertTo-Dockerfile` command to create a Dockerfile from the VHD.

	Copy and paste the command below into your Windows console window.

	```
	ConvertTo-Dockerfile -ImagePath c:\ws2016.vhd -Artifact IIS -OutputPath C:\windowstweetapp -Verbose
	```

	As mentioned before, Image2Docker will scan the VHD and create a Dockerfile based on the contents of the VHD. The list below explains the command-line arguments.

	* `ImagePath` specifies where the VHD can be found.

	* `Artifact` specifies what feature or code to look for.

	* `OutputPath` specifies where to write the Dockerfile and other items.

	* `Verbose` instructs the script to provide extra output.

When the process completes, you'll find a Dockerfile in `c:\windowstweetapp`.

### <a name="task3.2"></a> Task 3.2: Build and Push Your Image to Docker Trusted Registry

![]({{site.baseurl}}/images/windows75.png)

1. CD into the `c:\windowstweetapp` directory (this is where your Image2Docker files have been placed).

	```
	cd c:\windowstweetapp\
	```

2. As we did before, we're going to export some environment variables to make the commands that follow a bit more "Copy / Paste" friendly.

	1. Copy the DTR Hostname from the bottom of the main PWD screen.

	2. Assign it to the $DTR_HOST environment variable (put the value between quotes).

		```
		$DTR_HOST="<dtr hostname>"
		```

	3. Copy the user name from the bottom of the PWD screen (under credentials it's the first of the two listed).

	4. Assign it to the $USER environment variable (put the value in quotes).

		```
		$USER="<user name>"
		```

	5. Echo them back to make sure they are set properly.

		```
		echo $DTR_HOST $USER
		```

2. Use `docker image build` to build your Windows tweet web app image from the Dockerfile.

	Be sure to include the period (`.`) at the end of the command.

	```
	docker build -t $DTR_HOST/$USER/windows_tweet_app .
	```

	> **Note**: Feel free to examine the Dockerfile in this directory if you'd like to see how the image is being built.

	Your output should be similar to what is shown below.

	```
	PS C:\windowstweetapp> docker build -t $DTR_HOST/$USER/windows_tweet_app .

	Sending build context to Docker daemon  6.144kB
	Step 1/10 : FROM microsoft/windowsservercore
	 ---> 590c0c2590e4

	<output snipped>

	Step 10/10 : HEALTHCHECK CMD powershell -command     try {      $response = Invoke-WebRequest http://localhost -UseBasic
	Parsing;      if ($response.StatusCode -eq 200) { return 0}      else {return 1};     } catch { return 1 }
	 ---> Running in ab4dfee81c7e
	 ---> d74eead7f408
	Removing intermediate container ab4dfee81c7e
	Successfully built d74eead7f408
	Successfully tagged $DTR_HOST/$USER/windows_tweet_app:latest
	```

	> **Note**: It will take several minutes for your image to build.

4. Log into Docker Trusted Registry.

	```
	PS C:\> docker login $DTR_HOST
	Username: <your username>
	Password: <your password>
	Login Succeeded
	```

5. Push your new image to your Docker Trusted Registry.

	```
	PS C:\Users\docker> docker push $DTR_HOST/$USER/windows_tweet_app

	The push refers to a repository [<dtr hostname>/<your username>/windows_tweet_app]
	5d08bc106d91: Pushed
	74b0331584ac: Pushed
	e95704c2f7ac: Pushed
	669bd07a2ae7: Pushed
	d9e5b60d8a47: Pushed
	8981bfcdaa9c: Pushed
	25bdce4d7407: Pushed
	df83d4285da0: Pushed
	853ea7cd76fb: Pushed
	55cc5c7b4783: Skipped foreign layer
	f358be10862c: Skipped foreign layer
	latest: digest: sha256:e28b556b138e3d407d75122611710d5f53f3df2d2ad4a134dcf7782eb381fa3f size: 2825
	```

### <a name="task3.3"></a> Task 3.3: Deploy the Windows Web App

Now that we have our Windows Tweet App on the DTR server, let's deploy it.

The process is going to be almost identical to how we did the Linux version, with one small exception: Docker EE on Windows Server 2016 does not currently support ingress load balancing, so we'll expose the ports in `host` mode using `dnsrr`.

1. Switch back to UCP in your web browser.

2. In the left hand menu click `Services`.

3. In the upper right corner click `Create Service`.

4. Enter `windows_tweet_app` for the name.

5. Under `Image`, enter the path to your image which should be `<dtr hostname>/<your username>/windows_tweet_app`

	> **Note**: You can copy the image path from the output of your `docker push` command.

6. From the left hand menu click `Network`.

7. Set the `ENDPOINT SPEC` to `DNS Round Robin`. This tells the service to load balance using DNS. The alternative is VIP, which uses IPVS.

8. Click `Publish Port+`.

	We need to open a port for our web server. This app runs on port 80 which is used by DTR so let's use 8082.
9. Fill out the port fields as shown below. **Be sure to set the `Publish Mode` to `Host`**.

	![]({{site.baseurl}}/images/windows_ports.png)

10. Click 'Confirm'.

11. Click `Create` near the bottom right of the screen.

After a few seconds you should see a green dot next to your service name. Once you see you green dot you can  point your web browser to `http://<windows host>:8082` to see the running website.

## <a name="task4"></a> Task4: Deploying a Multi-OS Application

For our last exercise we'll use a Docker Compose file to define an application that uses a Java front end designed to be deployed on Linux, with a Microsoft SQL Server back end running on windows. We'll then deploy that application as a Docker Stack.

### <a name="task4.1"></a> Task 4.1: Examine the Docker Compose file

We'll use a Docker Compose file to define our application. With this file we can define all our services and their parameters, as well as other Docker primitives such as networks. In a later step we can deploy a Docker Stack from this file.

Let's look at the Docker Compose file:

```
version: "3.2"

services:

  database:
    image: sixeyed/atsea-db:mssql
    ports:
      - mode: host
        target: 1433
    networks:
     - atsea
    deploy:
      endpoint_mode: dnsrr

  appserver:
    image: mikegcoleman/atsea_appserver:1.0
    ports:
      - target: 8080
        published: 8080
    networks:
      - atsea

networks:
  atsea:
```

There are two services. `appserver` is our web frontend written in Java, and `database` is our Microsoft SQL Server database. The rest of the commands should look familiar as they are very close to what we used when we deployed our tweet services manually.

One thing that is new, is the creation of an overlay network (`atsea`). Overlay networks allow containers running on different hosts to communicate over a private software-defined network. In this case, the web frontend on our Linux host will use the `atsea` network to communicate with the database.

### <a name="task4.2"></a> Task 4.2 Deploy the Application Stack

A `stack` is a group of related services that make up an application. Stacks are a newer Docker primitive, and can be deployed from a Docker Compose file.

Let's Deploy a new application stack using the Docker Compose file above.

1. Move to the UCP console in your web browser.

2. In the left hand menu click `Stacks`.

3. In the upper right click `Create Stack`.

4. Enter `atsea` under `NAME`.

5. Select `Services` under `MODE`.

6. Select `SHOW VERBOSE COMPOSE OUTPUT`.

7. Paste the compose file from above into the `COMPOSE.YML` box.

8.  Click `Create`

	You will see some output to show the progress of the deployment, and then a banner will pop up at the bottom indicating the deployment was successful.

9. Click `Done`.

You should now be back on the Stacks screen.

1. Click on the `atsea` stack in the list.

2. From the right side of the screen choose `Services` under `Inspect Resource`.

	![]({{site.baseurl}}/images/inspect_resoource.png)

	Here you can see the two services running. It may take a few minutes for the database service to come up (the dot to turn green). Once it does, move on to the next section.


### <a name="task4.3"></a> Task 4.3: Verify the Running Application

1. To see the running web site (an art store), point your web browser to port 8088 on either of the Linux-based cluster nodes `http://<UCP hostname>:8080>`.

	The thumbnails you see displayed are actually pulled from the SQL database. This is how you know that the connection is working between the database and web front end.

--------

## Survey
We're grateful you've chosen to spend some time with us today. We'd appreciate the opportunity to hear from  you about what you liked and what we might improve.

If you could please fill out this very short survey, it'd be greatly appreciated.

Thanks, the DockerCon Hands-on Labs team.

[Click Here for the survey](https://docs.google.com/forms/d/e/1FAIpQLSfu5jatKvKLdGZviRQ7SdXaCKKUmLNYGz2-WyvQKVoIpGMaDQ/viewform?usp=pp_url&entry.413124055=Deploying+Multi-OS+Applications+with+Docker+EE&entry.879050699)

--------
In this lab we looked at three different ways to deploy Linux, Windows, and hybrid-OS applications on Docker EE.

If you're interested in continuing, you can pick up from right here and do the Application Lifecycle lab. If you choose to go straight into that lab, you can skip all the tasks in Part 1 and move right to Part 2.

Additionally, there are several other Docker labs available, so be sure to check those out.

You can find more information on Docker EE at [http://www.docker.com](http://www.docker.com/enterprise-edition) as well as continue exploring using our hosted trial at [https://dockertrial.com](https://dockertrial.com)
