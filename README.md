This script is based on golang's work: https://github.com/golang/build/tree/master/env/openbsd-amd64

make.bash creates a Google Compute Engine VM image to run the Go
OpenBSD builder, booting up to run the buildlet.

make.bash should be run on a Linux box with expect and qemu.
Debian packages: expect qemu-utils qemu-system-x86.

After it completes, it creates a file openbsd-amd64-gce.tar.gz

Then:
    gsutil cp -a public-read openbsd-amd64-gce.tar.gz gs://go-builder-data/openbsd-amd64-64-snap1.tar.gz
Or just use the web UI at:
    https://console.developers.google.com/project/symbolic-datum-552/storage/browser/go-builder-data/

Then:
   gcloud compute --project symbolic-datum-552 images delete openbsd-amd64-64-snap1
   gcloud compute --project symbolic-datum-552 images create openbsd-amd64-64-snap1 --source-uri gs://go-builder-data/openbsd-amd64-64-snap1.tar.gz

The VM needs to be run with the GCE metadata attribute "buildlet-binary-url" set to a URL
of the OpenBSD buildlet (cross-compiled, typically).

    buildlet-binary-url == https://storage.googleapis.com/go-builder-data/buildlet.openbsd-amd64