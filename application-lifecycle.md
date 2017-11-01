# Managing the Aplication Lifecycle with Docker EE

Docker EE 17.06 is the first Containers-as-a-Service platform to offer production-level support for the integrated management and security of Linux AND Windows Server Containers.

In this lab we'll deploy a multi-service application that includes both Windows and Linux components. Then we will then look at upgrades and rollback, scaling up, and how Docker EE handles system interruptions.

> **Difficulty**: Intermediate (assumes basic familiarity with Docker)

> **Time**: Approximately 30 minutes

> **Tasks**:
>

> * [Task 1: Deploy a Multi-OS Application](#Task 1)
>   * [Task 1.1: Add a Windows node to the cluster](#task1.1)
>   * [Task 1.2: Examine the Docker Compose File](#task1.2)
>   * [Task 1.3: Deploy the Application Stack](#task1.3)
>   * [Task 1.4: Verify the Running Application](#task1.4)
> * [Task 2: Application Lifecycle Management](#Task2)
>   * [Task 2.1: Upgrading the Web Front-end](#Task2.1)
>   * [Task 2.2: Scaling the Web Front-end](#Task2.2)
>   * [Task 2.3: Dealing with an Application Failure](#Task2.3)

## Understanding the Play With Docker Interface and Lab Environment

This lab makes use of a three node Docker EE cluster. The cluster is comprised of two Linux nodes and one Windows Server node. 

* The first Linux node acts as our Universal Control Plane (UCP) manager node (also referred to as the UCP server). 
	
* The second Linux node serves two purposes. It's a UCP worker node, and it also hosts Docker Trusted Registry (DTR) server. 
	
* The Windows Server node is deployed as a worker node in our cluster. 

All of this is presented via a web-based learning environment, Play with Docker (PWD). 

> **Note**: PWD is NOT part of Docker Enterprise Edition. It's a web framework for hosting Docker instances for use in training activities. 

![]({{site.baseurl}}/images/pwd_screen.png)

There are three main components to the PWD interface

### 1. Console Access
Play with Docker provides access to the 3 Docker EE hosts in your Cluster. These machines are:

* A Linux-based Docker EE 17.06 Manager node
* A Linux-based Docker EE 17.06 Worker node
* A Windows Server 2016-based Docker EE 17.06 Worker Node

By clicking a name on the left, the console window will be connected to that node.

### 2. Access to your Universal Control Plane (UCP) and Docker Trusted Registry (DTR) servers

Additionally, the PWD screen provides you one-click access to the Universal Control Plane (UCP) web-based management interface as well as the Docker Trusted Registry (DTR) web-based management interface. Click on either the `UCP` or `DTR` button will bring up the respective server web interface in a new tab.

### 3. Session Information

Throughout the lab you will be asked to provide either hosntnames or login credentials that are unique to your environment. These are displayed for you at the bottom of the screen.

## Document conventions

- When you encounter a phrase in between `<` and `>`  you are meant to substitute in a different value.

	For instance if you see `<dtr domain>` you would actually type something like `ip172-18-0-7-b70lttfic4qg008cvm90.direct.microsoft.play-with-docker.com`

- When you see the Linux penguin all the following instructions should be completed in your Linux console

	![]({{site.baseurl}}/images/linux75.png)

- When you see the Windows flag all the subsequent instructions should be completed in your Windows cosnsole.

	![]({{site.baseurl}}/images/windows75.png) 


## <a name="Task_1"></a>Task 1: Deploying a Multi-OS Application

For this lab we'll use a Docker Compose file to deploy an application that uses a Java front end designed to be deployed on Linux, with a Microsoft SQL Server back end running on windows.

Before we can deploy the application we will need to add a Windows node to our cluster.

1. Open the following link **in a new tab or window**: [PWD environment sign-in page](https://ee.microsoft.play-with-docker.com)

  > **Note**: You might want to right click the above link and open it in a new tab or window

2. Fill out the form, and click `submit`. You will then be redirected to the PWD environment.

3. Click `Access`

	It will take a few minutes to provision out your PWD environment.

### <a name="task1.1"></a>Task 1.1: Install a Windows worker node

1. From the main PWD screen click the `UCP` button on the left side of the screen

	> **Note**: Because this is a lab-based install of Docker EE we are using the default self-signed certs. Because of this your browser may display a security warning. It is safe to click through this warning.
	>
	> In a production environment you would use certs from a trusted certificate authority and would not see this screen.
	> ![]({{site.baseurl}}/images/ssl_error.png)

2. When prompted enter your username and password (these can be found below the console window in the main PWD screen). The UCP web interface should load up in your web browser.

	> **Note**: Once the main UCP screen loads you'll notice there is a red warning bar displayed at the top of the UCP screen, this is an artifact of running in a lab environment. A UCP server configured for a production environment would not display this warning
	>
	> ![]({{site.baseurl}}/images/red_warning.png)


3. From the main dashboard screen, click `Add a Node` on the bottom left of the screen

	![]({{site.baseurl}}/images/add_a_node.png)

4. Copy the text from the dark box shown on the `Add Node` screen.

	> **Note** There is an icon in the upper right corner of the box that you can click to copy the text to your clipboard
	> ![]({{site.baseurl}}/images/join_text.png)


	> **Note**: You may notice that there is a UI component to select `Linux` or `Windows`on the `Add Node` screen. In a production environment where you are starting from scratch there are a few prerequisite steps to adding a Windows node. However, we've already done these steps in the PWD envrionemnt. So for this lab, just leave the selecton on `Linux` and move on to the next step.


![]({{site.baseurl}}/images/windows75.png)

6. Switch back to the PWD interface, and click the name of your Windows node. This will connect the web-based console to your Windows Server 2016 Docker EE host.

7. Paste the text from the `Add Node` dialogue at the command prompt in the Windows console.

	You should see the message `This node joined a swarm as a worker.` indicating you've successfully joined the node to the cluster.

5. Switch back to the UCP server in your web browser and click the `x` in the upper right corner to close the `Add Node` window

6. You should be taken to the `Nodes` screen will will see 3 nodes listed at the bottom of your screen.

	After a minute or two refresh your web browswer to ensure that your Windows worker node has come up as `healthy`. Once this happens, you can move on to the next step.

	![]({{site.baseurl}}/images/node_listing.png)

### <a name="task1.2"></a> Task 1.2: Examine the Docker Compose file

We'll use a Docker Compose file to instantiate our application. With this file we can define all our services and their parameters, as well as other Docker primatives such as networks.

Let's look at the Docker Compose file:

```
version: "3.2"

services:

  database:
    image: dockersamples/atsea-db:mssql
    ports:
      - mode: host
        target: 1433
    networks:
     - atsea
    deploy:
      endpoint_mode: dnsrr

  appserver:
    image: dockersamples/atsea-appserver:1.0
    ports:
      - target: 8080
        published: 8080
    networks:
      - atsea

networks:
  atsea:
```

There are two services. `appserver` is our web frontend written in Java, and `database` is our Microsoft SQL Server database. The databased and web front end communicate over port `1433` and we'll expose port `8080` on the host to present the web front end. 

Additionally we are creating an overlay network (`atsea`). Overlay networks allow containers running on different hosts to communicate over a private software-defined network. In this case, the web frontend on our Linux host will use the `atsea` network to communicate with the database.

### <a name="task1.3"></a> Task 1.3: Deploy the Application Stack

A `stack` is a group of related services that make up an application. Stacks are a newer Docker primative, and can be deployed with a Docker Compose file.

Let's Deploy an application stack using the Docker Compose file above.

1. Move to the UCP console in your web browser

2. In the left hand menu click `Stacks`

3. In the upper right click `Create Stack`

4. Enter `atsea` under `NAME`

5. Select `Services` under `MODE`

6. Select `SHOW VERBOSE COMPOSE OUTPUT`

7. Paste the compose file from above into the `COMPOSE.YML` box

8.  Click `Create`

	You will see some output to show the progress of your deployment, and then a banner will pop up at the bottom indicating your deployment was successful.

9. Click `Done`

You should now be back on the Stacks screen.

1. Click on the `atsea` stack in the list

2. From the right side of the screen choose `Services` under `Inspect Resource`

	![]({{site.baseurl}}/images/inspect_resoource.png)

	Here you can see your two services running. It may take a few minutes for the database service to come up (the dot to turn green). Once it does move on to the next section.


### <a name="task1.4"></a> Task 1.4: Verify the Running Application

1. To see our running web site (an art store) visit click the `atsea_appserver` service from the list

2. On the right hand of screen click on the link under `Published Endpoints`

	The thumbnails you see displayed are actually pulled from the SQL database. This is how you know that the connection is working between the database and web front end.

## <a name="task5"></a> Task 2: Application Lifecycle Management

Now that we've deployed our application, let's take a look at some common tasks that admins need to do to keep their apps running and up-to-date. We'll start by upgrading the web front end, next we'll scale that service to meet demand, and then finally we'll see how to deal with the failur of a node in our UCP cluster.

### <a name="task2.1"></a> Task 2.1: Upgrading the Web Front-end

In this section we're going to first simulate a failed upgrade attempt, and see how to deal with that. 

The way we upgrade a running service is to update the image that service is based on. In this case the image we're going to upgrade to is broken. So when it's deployed UCP will pause the upgrade process, from there we can roll the application back to it's previous state.

1. Move back into Universal Control Plane

2. If your services are not currently displayed, click on `Services` from the left hand menu

3. Click on the `atsea_appserver` service from the list

4. On the left, under `Configure` select `Details`

	![]({{site.baseurl}}/images/service_details.png)

5. Under image, change the value to `dockersamples/atsea-appserver:2.0`

6. Click `Update` in the bottom right corner

7. Click on the `atsea_appserver` service from the list

8. The service indicator will turn from green to red, and if you look at the details on the right you'll see the `Update Status` go from `Updating` to `Paused`

	This is because the container that backs up the service failed to start up.

	![]({{site.baseurl}}/images/update_status.png)

9. From the right hand side click `Containers` under `Inspect Resource` and you will see some of the containers have exited with an error.

	Also notice under image that these errored out containers are running the `2.0` version of our application

	To recover from the failed upgrade, we're going to use Docker EE's rollback feature. 

10. Click `Services` from the left hand menu.

11. Click the `atsea_appserver` service from the list.

12. Under `Actions` on the right hand side click `Rollback`

	This will tell UCP to restore the service to its previous state. In this case, running the 1.0 version of our webserver image

	After a few seconds the indicator should go from red to green, when it does move on to the next step.

13. Click the `atsea_appserver` service from the list.

14. From the right hand side click `Containers` under `Inspect Resource` and you will see the container has started and is healthy.

	Also notice under image that the container is running the `1.0` version of our application.

15. In your web browser refresh thew website to ensure it's up and running. 

Now that we've dealt with a failed upgrade, let's look at rolling out a successful upgrade

1. Move back into Universal Control Plane

2. If your services are not currently displayed, click on `Services` from the left hand menu

3. Click on the `atsea_appserver` service from the list

4. On the left, under 'Configure` select `Details`

	![]({{site.baseurl}}/images/service_details.png)

5. Under image, change the value to `dockersamples/atsea-appserver:3.0`

6. Click `Update` in the bottom right corner

7. Click on the `atsea_appserver` service from the list

8. Notice the `Update Status` reads updating, and the indicator in the main area will go from green to red to green.

9.  From the right hand side click `Containers` under `Inspect Resource` and you will see the container has started and is healthy.

	Also notice under image that the container is running the `3.0` version of our application.

10. In your web browser refresh thew website to ensure it's up and running. Also notice the new color scheme that was implemented in Version 3 of the app. 

### <a name="task2.2"></a> Task 2.2: Scaling the Web Front-end

The new site design appears to have dramatically increased the popularity of your website. In order to deal with increased demand, you're going to need to scale up the number of containers in the `atsea_appserver` service.

1. Move to UCP in your web browser

2. From the left hand menu click `Services`

3. Click the `atsea_appserver` service

4. From the `Configure` drop down on the right choose `Scheduling`

5. Change `Scale` from `1` to `4`

6. Click `Update`

7. The indicator changes to yellow to indicate the service is still running, but not in our desired state (we want four containers, and there are currently fewer than that running). Regardless, your website is still available at this point.

	After a minute or so you'll see the indicator turn green, and you will see the status go to `4/4`

8. Click the `atsea_appserver` from the list

9. From the right hand side click `Containers` under `Inspect Resource` and you will see the four containers have started and are healthy.

	Also notice under `Node` that some containers are running on `worker1` and some are running on `manager1`

10. Go to your website in your brower and refresh the page, you will notice in the upper right the IP and Host change. This output is the IP and container ID of the actual container that served up the web page. You will see it cycle through the four containers backing up your service. 

	> **Note**: If you are not running in an incognito window you may need to force your browser to ignore the cache in order to see the values change. Consult the help section of your browser to learn how to do this.

Everything seems to be humming along nicely until one of your nodes in the cluster fails. In the next section we'll show how Docker EE deals with these sort of failuers.

### <a name="2.3"></a> Task 2.3: Dealing with an Application Failure

Docker EE will always try and reconcile your services to their desired state. For instance, in the case of our web frontend, we have specified we want four containers running. If for some reason the number ever drops below four, Docker EE will attempt to get the service back to four containers.

In this section we're going to simulate a node failure and see how Docker EE handles the situation. We're not actually going to crash a node. What we're going do do is put our worker node in `Drain` mode - which is essentially maintenance mode. We are telling Docker EE to shut all the containers that are running on that node down, and not schedule any additional work on to that node.

1. Move to UCP in your web browser

2. If the filter bar is active (the blue bar at the top of the screen) - click the `x` in the upper right corner to clear the filter.

3. From the left menu click `Nodes`

4. Click on `worker1`

5. From the `Configure` dropdown on the right side select Details

6. Under `Availability` click `Drain`

7. Click `Save`

	This will immediatley put the `worker1` node into Drain mode, and stop all running containers on that node.

8. Go to the AtSea website and refresh to verify it's still running.

	Even though one node failed, the built in Docker EE load balancer will direct traffic to the containers running on our healthy `manager1` node

9. Move back to UCP

10. Click the `x` in the upper right corner to close the `Edit Node` screen

	Notice that `worker1` still has a green indicator, this is because technically the node is still running and healthy. However, on the right hand side you'll see the `Availability` listed as `DRAIN`

10. Click on `Services` from the left hand menu

10. Click on the `atsea_appserver`

11. From the `Inspect Resource` drop down on the right select `Containers`

	Notice that the two containers that were running on `worker1` have been stopped, and they have been restarted on `manager1`

---

## Survey 
We're grateful you've chosen to spend some time with us today. We'd appreciate the opportunity to hear from  you about what you liked and what we might improve. 

If you could please fill out this very short survey, it'd be greatly appreciated. 

Thanks, the DockerCon Hands-on Labs team. 

[Click Here for the survey](https://docs.google.com/forms/d/e/1FAIpQLSfu5jatKvKLdGZviRQ7SdXaCKKUmLNYGz2-WyvQKVoIpGMaDQ/viewform?usp=pp_url&entry.413124055=Managing+the+Application+Lifecycle+with+Docker+EE&entry.879050699)

---

## Conclusion
In	this lab we've looked how Docker EE helps you deal with upgrades, scaling, and system failures.

There are several other Docker EE labs available, so be sure to check those out. 

You can find more information on Docker EE at [http://www.docker.com](http://www.docker.com/enterprise-edition) as well as continue exploring using our hosted trial at [https://dockertrial.com](https://dockertrial.com)
