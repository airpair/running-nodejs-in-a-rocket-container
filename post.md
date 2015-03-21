# Introduction 

The internet is abuzz with docker this, container that.  Rocket is another entry into the field of contanerizing your applications. It's very new and the spec is still under heavy development. It's already showing it can be a viable container runtime.

Things I enjoy most about rocket is the manditory crypyographic signing, as well as using proven web technologies to deliver container images, HTTP. This means we can easily host images on S3, CDN's or on our private clouds. It also comes with a well thought out pattern for image discovery using meta tags on a target domain.

While there has been lots published on running statically compiled golang applications in a coreos/rocket container, I noticed that the interwebs came up blank when looking for examples of running node.js. This is my attempt to resolve that.

There are several ways to determine what libs are required to run an application. You can use ldd and strace to hunt them all down manually.  Since this was supposed to be a quick exploration I reached for Jailkit. While its original intent is to help setup chroot jails it also happens to work perfectly for our current situation.

# Dependencies

I’ll wait here while you install the following tools needed for our journey:

- Jailkit
- actool
- rkt

Jailkit requires root so we’ll switch into root for the rest of the build process:

````
sudo su -
````

# Building your first image

Then we’ll create the folder structure that we will build our image / container into:

````
mkdir -p node-layout/rootfs
````

Next we’ll lean on jk_cp to copy node and its friends into this new structure we created.  This will automatically determine the shared libs we need to run node.js and move them into the expected places in our newly created container.  

Run this to make it so:

````
jk_cp /root/node-layout/rootfs /usr/bin/node
````

To create a container the spec requires us to create a /root/node-layout/manifest which is a json encoded file. The example below shows the required fields for a valid manifest:

````
{
  "acKind": "ImageManifest",
  "acVersion": "0.1.0",
  "name": "deedubs.com/node",
  "app": {
    "exec": [
      "/usr/bin/node /server.js"
    ]
  }
}
````

Next we’ll use the example from the node.js homepage, saving it to /root/node-layout/rootfs/server.js

````
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, '127.0.0.1');
console.log('Server running at http://127.0.0.1:1337/');
````

At this point we’re ready to compile the image using actool:

````
actool --debug build node-layout/ example-node.aci
````

# Image validation

The image we just generated can be validated using the actool again with the command:

````
actool --debug validate example-node.aci
example-node.aci: valid app container image
````

# Blast off!

As long as everything above was successful we are ready to launch our first rocket container!

Let's do this!

````
rkt --debug run example-node.aci
````

Currently rocket is a prototype but its already showing A LOT of promise and I’m excited adopt it into the infrastructure running PineHQ!