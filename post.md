## Introduction 

The internet is abuzz with Docker this, container that. Rocket is another entry into the field of containerizing your applications. It's very new and the spec is still under heavy development. It's already showing it can be a viable container runtime.

Things I enjoy most about Rocket include the mandatory cryptographic signing, as well as using proven web technologies like HTTP to deliver container images. This means we can easily host images on Amazon S3, CDNs or on our private clouds. It also comes with a well-thought-out pattern for image discovery using meta tags on a target domain.

While there has been lots published on running statically compiled golang applications in a Rocket container, I noticed that the interwebs came up blank when looking for examples of running node.js. This is my attempt to resolve that.

There are several ways to determine what libraries are required to run an application. One of the simplest is just using tools like `ldd` and `strace` to hunt them all down manually. Since this was supposed to be a quick exploration I reached instead for [Jailkit](http://olivier.sessink.nl/jailkit/). While its original intent is to help set up chroot jails, it also happens to work perfectly for our current situation.

## Dependencies

I’ll wait here while you install the following tools needed for our journey:

- Jailkit
- actool
- rkt

Jailkit requires superuser permissions, so we’ll switch to the root user for the rest of the build process:

```bash
sudo su -
```

## Building your first image

Then we’ll create the folder structure that we will build into our image:

```bash
mkdir -p node-layout/rootfs
```

Next we’ll lean on `jk_cp` to copy node and its friends into this new structure we created. This will automatically determine the shared libs we need to run node.js and move them into the expected places in our newly created container.  

Run this to make it so:

```bash
jk_cp /root/node-layout/rootfs /usr/bin/node
```

To create an Application Container Image, the spec requires us to create a `/root/node-layout/manifest` which is a simple JSON-encoded file. The example below shows the required fields for a valid manifest:

```javascript
{
  "acKind": "ImageManifest",
  "acVersion": "0.4.0",
  "name": "deedubs.com/node",
  "app": {
    "exec": [
      "/usr/bin/node /server.js"
    ]
  }
}
```

Next we’ll use the example from the [node.js](https://nodejs.org/) homepage, saving it to `/root/node-layout/rootfs/server.js`

```javascript
var http = require('http');
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello World\n');
}).listen(1337, '127.0.0.1');
console.log('Server running at http://127.0.0.1:1337/');
```

At this point we’re ready to compile the image using `actool`:

```bash
actool --debug build node-layout/ example-node.aci
```

## Image validation

The image we just generated can be validated using `actool` again with the command:

```bash
actool --debug validate example-node.aci
example-node.aci: valid app container image
```

## Blast off!

As long as everything above was successful we are ready to launch our first Rocket container!

Let's do this!

```bash
rkt --debug run example-node.aci
```

Currently Rocket is a prototype but it's already showing A LOT of promise and I’m excited adopt it into the infrastructure running PineHQ!
