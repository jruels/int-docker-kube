## Union Filesystems

Use overlayfs to mount directories and full filesystems

### Overlay two directories

Create a directory to work within. In this example, the directory is `~/mysandbox`.

```bash
mkdir ~/mysandboxdir && cd ~/mysandboxdir
```

Create directories representing the lower, upper, working, and destination merged directories

```bash
mkdir work layer1 layer2 merged
```

These directories will be used throughout this lab to demonstrate how overlayfs merges multiple
directory trees to present a single, unified directory.

First, let's break down each of the directories.

* The layer1 directory represents the `lowerdir` in the mount command
  * Files and directories created here will be present in the `merged` directory
* The layer2 directory represents the `upperdir` in the mount command
  * Files and directories that exist in both `upperdir` and `lowerdir` will take the contents of `upperdir` as priority
  * Any file operations that occur in the `merged` directory will be copied here
* the work directory represents a location where files are prepared before becoming visible in the merged directory
* the merged directory represents the final result of merging layer1 and layer2 using the rules above

Let's now use these directories to create our own overlay mount.
Create a mount of type `overlay`, providing arguments defining the desired lower, upper, and merged directory

```bash
sudo mount -t overlay overlay -o lowerdir=./layer1,upperdir=./layer2,workdir=./work merged
```

Touch a file in `./layer1`. We'll use this file to demonstrate how certain I/O scenarios affect the layers.

```bash
touch ./layer1/foo
```

Notice that a file named `foo` now exists in `./merged`.

Now, touch a file in `./merged`.

```bash
touch ./merged/bar
```

List files in the `./layer2` directory and you will now see a file called `bar` exists.

Let's make a change to the `./merged/foo` file. This was originally copied from the `./layer1` directory.

```bash
echo "hello" > ./merged/foo
```

Now, inspect the contents of `./layer1/foo`. Notice anything missing?  Where did the write to `foo` go?

Upon further investigation you will notice that the write into `./merged/foo` is not in `./layer1`, but it exists
in both the `merged` AND `layer2` directories, since `layer2` is defined as the upperdir in our mount statement.

Lastly, we'll delete a file in `./merged` that exists in `./layer1`.

```bash
echo "hi" > ./layer1/baz
rm ./merged/baz
```

What are the contents of `./layer1`? Does `baz` exist there?
What about `./layer2`? Is `baz` there?

You probably found something like this in `./layer2`

```bash
$ ls -l layer2
-rw-r--r-- 1 1 root root 0, 0 Sep 19 04:45 bar
c---------     root root    0 Sep 19 04:33 baz
-rw-r--r-- 1   root root   22 Sep 19 04:34 foo
```

When a file is deleted from `./merged` that exists in `./layer1`, a _whiteout_ character device file is generated and stored in `./layer2`.
This instructs overlayfs to ignore `./layer1/baz` and ensure the whiteout file is not visible in `./merged`.

### Overlaying Multiple Layers

Overlayfs allows you to specify multiple `lowerdir` directories. The specified lower directories will be stacked beginning from the rightmost one and going left.

Let's add a `./layer0` directory and recreate our overlayfs mount with layer0 as the lowest priority.

```bash
mkdir ./layer0
sudo mount -t overlay overlay -o lowerdir=./layer0:./layer1,upperdir=./layer2,workdir=./work merged
```

Notice lowerdir = `./layer0:./layer1` in the mount command.

Create files in `./layer0` and `./layer1` to see results in `./merged`

```bash
echo "Hello from Layer 0" > ./layer0/qux
echo "Hello from Layer 1" > ./layer1/qux
cat ./merged/qux
```

What's in `./merged/qux`? Re-run the mount command with the lower layers reversed and inspect `./merged/qux` again.

```bash
sudo mount -t overlay overlay -o lowerdir=./layer1:./layer0,upperdir=./layer2,workdir=./work merged
```

When the directory order is reversed, a new version of `qux` takes precedence and now exists in the `merged` directory.

### Docker and Overlayfs

Now, let's see how docker uses the overlay2 driver to persist images on a docker host.

Make sure we have an image available on the local docker host by grabbing the redis:4.0.11 image.

```bash
docker pull redis:4.0.11
```

Docker has an `inspect` command that can be used to retrieve a json representation of an image's metadata.
Let's use this command to discover the layers that make up the redis image, as well as the effective command that will be used to mount and render the image's filesystem.

```bash
docker inspect redis:4.0.11 | jq -r '.[].GraphDriver'
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/8cc191d7e8944039a2ddb8cc17fc30f6bfa0aa8efe77706b987f381daf5561dc/diff:/var/lib/docker/overlay2/526988f6a76344f880e62dc5fd84263ee753ad0ead2dbb1417d31bf2a90267af/diff:/var/lib/docker/overlay2/91319dd0de72b2a4f6268959ee75c245b28cd1ddda3f1fb6833493ab759afc65/diff:/var/lib/docker/overlay2/8a9e2f0ebf929840f3130c9ad3e17d051f3c1ea0c54acbc56ff888cc8fb59838/diff:/var/lib/docker/overlay2/1812eed1e977a42f10daa960a16b19e96503f88c02529bc370900851a9c45df9/diff",
    "MergedDir": "/var/lib/docker/overlay2/4b3028a6bd702ced2ef2373f478443a55192bb0b13353dde89eccf1050b3ae14/merged",
    "UpperDir": "/var/lib/docker/overlay2/4b3028a6bd702ced2ef2373f478443a55192bb0b13353dde89eccf1050b3ae14/diff",
    "WorkDir": "/var/lib/docker/overlay2/4b3028a6bd702ced2ef2373f478443a55192bb0b13353dde89eccf1050b3ae14/work"
  },
  "Name": "overlay2"
}
```

You'll find the naming convention similar to that of the actual `mount` command you ran in the earlier exercise. If the running container opens a file for read or write,
that operation will occur in the `/var/lib/docker/overlay2/4b3028a6bd702ced2ef2373f478443a55192bb0b13353dde89eccf1050b3ae14/merged` directory.
