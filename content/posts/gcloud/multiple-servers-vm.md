+++
date = '2023-01-14T00:00:00-04:00'
draft = false
title = 'Using Google Cloud to host multiple servers on one VM'
+++

This blog post presents a solution to the following problem: we want to host two services, and have `test1.example.com` and `test2.example.com` route to them respectively. These services should run on one machine, on different ports. Google Cloud Load Balancing makes this relatively easy, although nginx solutions exist.

## Step 1: Set up a VM

We’ll create a VM with default settings. Create two directories, test1 and test2 inside the VM. Within both, we’ll create the same file called `server.js`.

```js
#!/usr/bin/env node
const express = require('express')
const app = express()
const port = 4000

app.get('/*', (req, res) => {
  res.send('Hello! You have reached Test Server 2.')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

We need one change: in `test1/server.js`, change the port to 3000, and change the `res.send` line to say Test Server 1. In both, run:

```sh
npm i -D express
```

Then in any directory, run

```sh
sudo npm i -g forever
```

In each directory, run

```sh
forever start server.js
```

We’re using `forever` to daemonize the processes, though you can do this using systemd as well. As of now, you have two services running, one on port 3000 and one on port 4000, and they will not stop running.

## Step 2: Set up the load balancer

iBefore setting up the load balancer, we’ll need to add our instance to an instance group. This is relatively straightforward: go to Compute Engine – Instance Groups, and set the type on the left to an Unmanaged instance group. There is a key part here, though–we need to use Named Ports. Add two: one called `port3000`, matched to port 3000, and another called `port4000` matched to port 4000. Time to set up the load balancer.

Go to Network Services – Load balancing in the left pane of Google Cloud. Create a load balancer, and select HTTPS as the type. We’ll add two frontend configurations: one for HTTP, and one for HTTPS. Create a static IP in the IP address box and use the same one for both frontends. For HTTPS, we’ll use a Google-managed SSL certificate.

Click on Backend services. We’ll need to create two here. Click on Backend services and backend buckets, and click on Create Backend Service. You can name these whatever you want–I’ve named them `test1` and `test2`. Set the backend type as Instance group, and select the instance group you just created. For our purposes, we’ll disable Cloud CDN. We’ll be prompted for a named port, and select `port3000`. Set the maximum utilization to 100%. Under Health Checks, click on Create Health Check. Give it a name–I called it `test1`. Make sure the protocol is TCP, and set the port to 3000. Click Save. You’ll do the same for the second backend service–except, use the `port4000` named port, and in Health Checks, set the port to 4000.

Next, click on Routing Rules. Use the Simple host and path rule option, and add the following rules:
* test1.example.com mapping to path /* and backend test1.
* test2.example.com mapping to path /* and backend test2.

Finally, create the load balancer. However, we’re not done yet.

## Step 3: Miscellaneous tasks

We have two more things to do. First, the instance’s firewall will not allow traffic to ports 3000 or 4000. This is an easy fix–go to VPC network – Firewall, and then Create Firewall Rule. I called it `port3000-port4000`, set targets to All instances in the network, source IP ranges to 0.0.0.0/0. Check TCP, and in ports, type 3000, 4000. Click Create. Go to Compute Engine – VM instances, select your instance, and click Edit. Under Network tags, add `port3000-port4000` and save.

Finally, all that’s left is DNS. Go to your domain registrar’s DNS settings, and add two A records: one for test1.example.com and one for test2.example.com. Have both records point to the IP address of the load balancer. To find this IP address, go to VPC network – IP addresses, and you’ll find the name of the IP address you created in the first step of the load balancer creation.

You’re done! You should now be able to navigate to test1.example.com and test2.example.com and have the two messages printed out.
