# Updating the Docker for AWS AMI

This is a short guide to using the public ami of moby linux to create a
new, patched version.

Please note: I'm a big believer in the "don't trust anybody" philosophy.
As such, this guide provides each and every step that I did to generate
the updated image so you don't have to take anything I say as blessed. I
will, however, also provide some shortcuts if that's easier for you and
you think I'm a trustworthy gal :)

## Compiling the Kernel

### Getting the current config

The first step is to find out what the current config is. I have
provided this in this repo `4.9.49-moby.config` but as I said: don't
take my word for it. You can find it out for yourself:

SSH into one of your docker managers and run:

```
gunzip -c /proc/config.gz > /tmp/$(uname -r).config
```

This will create `/tmp/4.9.49-moby.config`, copy this (as I say, it
should be the same as the one I've provided)

### Create a work server

We need a seperate server to work with the root volume, download the
linux source files, and compile them so I'd recommend making a tempoary
one just for this.

Pretty much any server will work here, so I'm not going to be too
specific, but I launched t2.nano Amazon Linux Instance.

You may want to give it a name (I called it "work server").

### Do the compilation

SSH into it and install the tools we'll need:

```
sudo yum install make gcc elfutils-libelf-devel
```

Download, exact and change directory into the latest copy of linux:

```
wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.14.13.tar.gz
tar xf linux-4.14.13.tar.gz
cd linux-4.14.13
```

Use your favourite editor to open up a new file called `.config` and
paste in the contents of `4.9.49-moby.config`.

Now, because that config that we had is for an old version of linux, we
need to tell it to update it to make it ready for the newer kernel,
which has more options. We can do this with:

```
make olddefconfig
```

This tells the Kernel Config system to update an old file using the
default values for any new config items.

Compile it:

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
managers / workers. It's the item called AMI ID and will look like:

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

Although the Docker for AWS Image (AMI) is publicly available, the
snapshot that is used to create each instance's root volume is not. If
it were public, we could just create a volume from that, but we have to
do it in a more roundabout way.

Head into your AWS console > EC2 > AMIs. Select "Public Images" and
search for your the ami id that we found in the step above.

Click it and launch it, this is going to be thrown away as soon as we
have the volume, so just pick a t2.nano (or if you're still free tier
eligible: t2.micro).

None of the config on this instance really matters, as we're going to be
shutting it down anyway, so just click next then go straight to
"Review and Launch" and Launch it. (You may want to give it a name to
make finding it later easier, otherwise just keep note of its instance
id)

Now immediately go to your instances page and stop it. You may get an
error to say it hasn't finished starting yet, but just keep clicking
"Yes, Stop" until it lets you. The reason we want to do this is because
we're going to detach the volume from it anyway.

Once its stopped, go to the instance information and click on
`/dev/xvdb` next to "Root Device". In the popup, click the `vol-...`
link to view the volume for this instance.

Right click the volume and detatch it. This should only take a second,
but you may need to refresh to see that it has finished. Once it has
done right click it again to attach it to your work server that we
created above. The device name doesn't matter particularly, it is likely
to be `/dev/sdf`.

### Copy the new kernel onto the volume

Back in your SSH terminal on the worker server, we have wait for the
compilation to finish. Once it has done, let's copy the new linux kernel
onto the device.

Mount the docker-for-aws volume that we attached in the previous step:

```
sudo mount /dev/xvdf1 /mnt
```

Copy your compiled kernel onto the volume:

```
sudo cp ~/linux-4.14.13/arch/x86/boot/bzImage /mnt/vmlinuz64
```

We now have a clean and updated volume, now unmount it again:

```
sudo umount /mnt
```

### Create the AMI

Back in the volume view on the AWS Console, right click and detach the
volume again. Again, once it's done and you've refreshed, right click
to attach it. This time we'll attach it to the docker node we created
and immediately shut down.

This time the device name *is* important, as Docker for AWS has been
configured to use a specific one. Enter `/dev/xvdb`.

Head over to the shut down docker instance, right click it and choose
Image > Create Image. The name and description are up to you, but in
the table, **make sure you tick the "Delete on Termination"** checkbox.

Hurray you have a new AMI!
