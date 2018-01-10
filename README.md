# Updating the Docker for AWS AMI

This is a short how-to guide for creating a copy of Moby Linux with an
updated kernel to protect against Meltdown.

## Preface and Notes

This guide will update you from kernel 4.9.49 to 4.14.13 which, most
notably has Kernel Page Table Isolation turned on to mitigate against
Meltdown.

Please note: I'm a big believer in the "don't trust anybody" philosophy;
don't take anything anyone says as blessed, especially me. This guide
provides each and every step that I did to generate the updated image so
you can audit it yourself and (hopefully) see I'm not trying to trick
you into installing malicious stuff on your systems.

## Compiling the Kernel

### Getting the current config

The first step is to find out what the current kernel config is. If you
can't be bothered to do this step, I have provided the config in this
repo (file: `4.9.49-moby.config`) but as I said: don't
take my word for it when you can easily find it out for yourself:

SSH into one of your docker managers. Now we're going to read the
current kernel config, as this is compressed, we just need to run it
through `gunzip` to decompress it.

```
gunzip -c /proc/config.gz > /tmp/$(uname -r).config
```

This will create `/tmp/4.9.49-moby.config`, copy this (it should be the
same as the one I've provided). Open it up and copy its contents.

### Create a work server

Moby Linux writes a bunch of stuff to the root device when it starts up,
and its not easy to delete that from within the machine without breaking
everything, so we're going to need a separate server to work with the
root volume instead.

In this server we'll also download the Linux source files and compile
them. Pretty much any server will work here, so I'm not going to be too
specific, but I launched t2.nano Amazon Linux Instance (if you want to
launch something else you can, but my next step assumes you have the
`yum` package manager, which is already installed on Amazon Linux.

You may want to give it a name (I called mine "work server" but this is
entirely up to you - it's just to make it easier to find when we jump
around in this guide).

### Do the compilation

SSH into the work server and install the tools we'll need:

```
sudo yum install make gcc elfutils-libelf-devel
```

Download, exact and change directory into the latest copy of Linux:

```
wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.14.13.tar.gz
tar xf linux-4.14.13.tar.gz
cd linux-4.14.13
```

Use your favourite editor to open up a new file called `.config` and
paste in the contents of `4.9.49-moby.config` that we found in the first
step.

Now, because the config that we had is for an old version of Linux, we
need to tell it to update it to make it ready for the newer kernel,
which has more options. We can do this with:

```
make olddefconfig
```

This tells the Kernel config system to update an old file using the
default values for any new config items.

Compile the kernel, telling it we're interested in the compressed
version and no modules:

```
make bzImage
```

This will take a while to compile, while it's going you can continue to
the next steps.

## Creating a new Image

### Find the current image

First we need to find out which AMI your stack is using. This is
different depending on what region you are starting instances in. You
can find this out by looking at the info on one of your current Docker
managers / workers. It's the item called "AMI ID" and will look like:

```
Moby Linux 17.09.0-ce-aws1 stable (ami-xxxxxxxx)
```

We're interested in the `ami-xxxxxxxx` part. Here's also a table of
values for you to check against:

| Region         | AMI ID       |
| ---------------|--------------|
| ap-northeast-1 | ami-3d940d5b |
| ap-northeast-2 | ami-61bd1d0f |
| ap-south-1     | ami-b3c793dc |
| ap-southeast-1 | ami-4ef18132 |
| ap-southeast-2 | ami-99a75afb |
| ca-central-1   | ami-2afc794e |
| eu-central-1   | ami-70f7651f |
| eu-west-1      | ami-7abe2e03 |
| eu-west-2      | ami-7b81991f |
| sa-east-1      | ami-a72466cb |
| us-east-1      | ami-6ebce314 |
| us-east-2      | ami-2eac874b |
| us-west-1      | ami-42f3f322 |
| us-west-2      | ami-98bc08e0 |


### Getting a fresh copy of the snapshot

Next we need a fresh copy of the original Moby Linux's root.
Unfortunately, although the Docker for AWS Image (AMI) is publicly
available, the snapshot that is used to create each instance's root
volume is not. If it were public, we could just create a volume from
that, but as it's not we have to do it in a more roundabout way.

Head into your AWS console > EC2 > AMIs. Select "Public Images" and
search for the AMI ID that we found in the step above.

Launch it; this instance is never going to run, we just want it for its
volume, so just pick a t2.nano and go straight to
"Review and Launch" and Launch it. (You may want to give it a name to
make finding it later easier, otherwise just keep note of its instance
id once its launched)

Now immediately go to your instances page and stop it. You may get an
error to say it hasn't finished starting yet, but just keep clicking
"Yes, Stop" until it lets you.

Once its stopped, go to the instance information and click on
`/dev/xvdb` next to "Root Device". In the pop up, click the `vol-...`
link to view the volume for this instance.

Right click the volume and detach it. This should only take a second,
but you may need to refresh to see that it has finished.

Next, right click it again and choose attach. This time we'll add it to
our work server that's busy compiling Linux for us. The device name
doesn't matter particularly, it is likely to default to `/dev/sdf`.

### Copy the new kernel onto the volume

Head back to the SSH terminal on the work server. Unfortunately at this
point we now need to wait for the compilation to finish. Once it has
done, we'll copy the new Linux kernel onto the device.

Mount the volume that we attached in the previous step so we can see its
files:

```
sudo mount /dev/xvdf1 /mnt
```

Copy your compiled kernel onto the volume, replacing the old one:

```
sudo cp ~/linux-4.14.13/arch/x86/boot/bzImage /mnt/vmlinuz64
```

Job done, now unmount it again:

```
sudo umount /mnt
```

### Create the AMI

Back in the volume view on the AWS Console, right click and detach the
volume again once again. Once it's done and you've refreshed, right
click to attach it again. This time we'll attach it back where it came
from: the docker node we created and immediately shut down.

This time the device name *is* important, as Docker for AWS has been
configured to use a specific one. Enter `/dev/xvdb`.

Head over to that shut down docker instance, right click it and choose
Image > Create Image. The name and description are up to you ("Moby
Meltdown" or "New Docker" or some variant perhaps?). In
the table, **make sure you tick the "Delete on Termination"** checkbox.

Hurray you have a new AMI!

### Cleanup

Terminate the two instances you made, and then delete the volume we were
working with. These are no longer needed.

## Setting up your stack to use your new image

### Make new Launch Configurations

Head on over to your "Launch Configurations" page. You should see two in
there:

```
[STACK NAME]-NodeLaunchConfig17090ceaws1-1[RANDOM GARBAGE]
[STACK NAME]-ManagerLaunchConfig17090ceaws1-1[RANDOM GARBAGE]
```

For each of these, we need to do the following steps.

It's not possible to just amend a launch configuration, instead we have
to create a new one. We can, however, use a copy of the original as a
starting point. Right click and choose "Copy launch configuration";
it will now take you to a screen that looks similar to the launch
instance pages.

You'll notice that "Moby Linux 17.09.0-ce-aws1 stable" is the selected
AMI, so we need to change that to our new one with "Edit AMI" on the
right. In the search page, go to My AMIs and search for the AMI you
created in the previous step. When you select this AMI, Amazon will warn
you that some things will reset; although this is annoying, we need that
AMI changed, so choose "Yes" and continue.

On the Instance Type page, this should have remembered what instance
type you already had setup, so don't change anything (unless you want a
different size of instance than you had before; feel free to do that, it
won't break anything) and click next.

On the next page, keep all the config the same, although you may want to
give the launch configuration a nicer name (I opted simply for "worker"
and "manager"), and click next.

The volumes should all be set up correctly, with the exception that the
size of the root volume has reset down to 1GB which is too small, change
that to whatever size volume you want and click next.

On the security group page **do not change anything**; it should have
remembered all of the settings from the Docker for AWS launch
configuration and changing this page can easily break your swarm. Click
review and finally create the Launch Configuration with your SSH key of
choice.

### Use these launch configurations

Go to your "Auto Scaling Groups" page, you should see two in there:

```
[STACK NAME]-NodeAsg-[RANDOM GARBAGE]
[STACK NAME]-ManagerAsg-[RANDOM GARBAGE]
```

For each one edit it and simply change the Launch Configuration to the
relevant one you created in the previous step.

From this point on, any *new* nodes or managers started to your swarm
should use the updated Linux kernel.

## TODOs

* Clean way of restarting / updating the old nodes
* Update the rest of the OS too (as it's on Alpine 3.5, which is pretty
old)
* Missing nfs_layout_flexfiles module, so the kernel no longer supports
NFS FlexFiles. Can be fixed alongside updating the OS.


## Disclaimer

THE GUIDE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
GUIDE OR THE USE OR OTHER DEALINGS IN THE GUIDE.
