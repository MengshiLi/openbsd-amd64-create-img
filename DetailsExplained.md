Google Cloud Platform has no default support for OpenBSD, neither does the OpenBSD community. 
The basic idea is generating a GCP-compatible image using QEMU, uploading it to GCP, then instantiating a VM with this image.

The starting point would be this [script](https://github.com/golang/build/tree/master/env/openbsd-amd64), based on which, some customized recipe can be further added.

### Create a GCP compatible image
The above [script](https://github.com/golang/build/tree/master/env/openbsd-amd64) mainly finishes the following steps.
#### Downloading OpenBSD ISO...
#### prepare some configuration files to be put into site66.tgz [ref](https://eradman.com/posts/autoinstall-openbsd.html):

> OpenBSD allows for custom software to be installed by adding a site-specific tgz file. If index.txt includs the new file it will appear in the menu; we only need to select the new package in *install.conf
> Set name(s) = site66.tgz
> To make this easy I am allowing for this one package to be installed without being signed.
> Checksum test for site66.tgz failed. Continue anyway = yes
> Unverified sets: site66.tgz. Continue without verification = yes

  - install.site
  - etc/installurl
  - etc/rc.local:  
> The script /etc/rc.local is for use by the system administrator. It is traditionally executed after all the normal system services are started, at the end of the process of switching to a multiuser runlevel. You might use it to start a custom service, for example a server that's installed in /usr/local. Most installations don't need /etc/rc.local, it's provided for the minority of cases where it's needed.
  - etc/sysctl.conf

#### prepare following files to be baked into ISO file.
  - auto_install.conf...
  - disklabel.template...
  - boot.conf
  - random.seed

#### Hack the ISO by baking above files into corresponding path.
  -  /6.6/amd64/site66.tgz
  -  /auto_install.conf
  -  /disklabel.tempate
  -  /etc/boot.conf
  -  /etc/random.seed
  - cmd: `Executing 'genisoimage -C 16,226544 -M /dev/fd/3 -l -R -graft-points /6.6/amd64/site66.tgz=site66.tgz /auto_install.conf=auto_install.conf /disklabel.template=disklabel.template /etc/boot.conf=boot.conf /etc/random.seed=random.seed | builtin_dd of=install66-amd64-patched.iso obs=32k seek=14159'` 

#### create initial disk.raw with qemu-img.
  - boot openbsd from iso created above, with disk.raw as the disk.
  - after boot, enter shell to copy auto_install.conf, disklabel.template from cdrom to /.
  - then exit, qemu will auto install openbsd into disk.raw, following the auto_install.conf
  - patch the system as specified at `install.site`, also install some necessary packages: bash/curl/git.

#### Some must-have setups in the script
  - `allow root ssh login` = yes
  - `com0` = yes

### With this image:
- upload it to google storage bucket, with `gsutil cp` 
`gsutil cp -a public-read openbsd-amd64-gce.tar.gz gs://go-builder-data/openbsd-amd64-64-snap1.tar.gz` or [WebUI](https://console.developers.google.com/project/symbolic-datum-552/storage/browser/go-builder-data/)
- convert it to a GCP-compatible image: 
```
gcloud compute --project symbolic-datum-552 images delete openbsd-amd64-64-snap1
gcloud compute --project symbolic-datum-552 images create openbsd-amd64-64-snap1 --source-uri gs://go-builder-data/openbsd-amd64-64-snap1.tar.gz
```
- instantiate a VM with this image:
- To run the OS, need to set custom metadata:
`buildlet-binary-url == https://storage.googleapis.com/go-builder-data/buildlet.openbsd-amd64`


### Reinstall the whole OS:
You can use the OS from that image directly. Here we reinstalled Openbsd as resizing the partition fails.
- `> boot bsd.rd`
- Partition as following:
  - use auto then 'e'dit 
  - delete /home 
  - /usr/local has 8-10GB 
  - create 2GB for /usr/src, /usr/obj, /usr/xenocara, /usr/xobj 
  - No worry for this: create 10GB for /cvs *unless* you just want to work off the unofficial git repository (might be easier) 
  - alloc the rest to /home
  - if you are unable to expand the disk to the full physical disk, you have to double check the OpenBSD area (the 'b' command in disklabel)
  - example for 100G disk
![IMAGE](quiver-image-url/CB2E9CDD05A95F2B2019085DE1EA252A.jpg =938x301)
- bsd.rd contains no set file, install from HTTP. issue `?` for server list
- Select set, remove game and x-display: `-game*` `-x*`
- Add some basic dev tools: `pkg_add screen git vim python glib2`


### Virtualbox
Someone also tried to use VirtualBox to create the image, as shown in this [guide:](https://dev.to/nabbisen/openbsd-vm-on-gcegcp---12-local-part-gp3), which however, doesn't work for me.
