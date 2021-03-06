# Installing Moonfire NVR

This document describes how to install Moonfire NVR on a Linux system.

## Downloading

See the [github page](https://github.com/scottlamb/moonfire-nvr) (in case
you're not reading this text there already). You can download the bleeding
edge version from the command line via git:

    $ git clone https://github.com/scottlamb/moonfire-nvr.git

## Building from source

There are no binary packages of Moonfire NVR available yet, so it must be built
from source. To do so, you can follow two paths:

* Scripted install: You will run some shell scripts (after preparing one or
  two files, and will be completely done. This is by far the easiest option,
  in particular for first time builders/installers. This process is fully
  described. Read more in [Easy Installation](easy-install.md)
* Manual build and install. This is explained in detail in these
  [instructions](install-manual.md)

Regardless of the approach for setup and installation above that you choose,
please read the further configuration instructions below.

## Further configuration

Moonfire NVR should be run under a dedicated user. It keeps two kinds of
state:

   * a SQLite database, typically <1 GiB. It should be stored on flash if
     available.
   * the "sample file directories", which hold the actual samples/frames of
     H.264 video. These should be quite large and are typically stored on hard
     drives.

Both states are intended to be accessed by moonfire-nvr only, but can be
changed after building. See below.
(See [schema.md](schema.md) for more information.)

The database changes and sample file directory changes require the moonfire-nvr
binary to be built, so can only be done after completing the build. The other
settings and preparations should be done before building.
Manual commands would look something like this:

    $ sudo addgroup --system moonfire-nvr
    $ sudo adduser --system moonfire-nvr --home /var/lib/moonfire-nvr
    $ sudo mkdir /var/lib/moonfire-nvr
    $ sudo -u moonfire-nvr -H mkdir db sample
    $ sudo -u moonfire-nvr moonfire-nvr init

### <a name="drive mounting"></a>Camera configuration and hard drive mounting

If a dedicated hard drive is available, set up the mount point:

    $ sudo vim /etc/fstab
    $ sudo mount /var/lib/moonfire-nvr/sample

Once setup is complete, it is time to add camera configurations to the
database. If the daemon is running, you will need to stop it temporarily:

    $ sudo systemctl stop moonfire-nvr

You can configure the system's database through a text-based user interface:

    $ sudo -u moonfire-nvr moonfire-nvr config 2>debug-log

In the user interface,

 1. add your sample file dirs under "Edit cameras and retention"
 2. add cameras under the "Edit cameras and streams" dialog.
    There's a "Test" button to verify your settings directly from the dialog.
    Be sure to assign each stream you want to capture to a sample file
    directory.
 3. Assign disk space to your cameras back in "Edit cameras and retention".
    Leave a little slack (at least 100 MB per camera) between the total limit
    and the filesystem capacity, even if you store nothing else on the disk.
    There are several reasons this is needed:

       * The limit currently controls fully-written files only. There will be up
         to two minutes of video per camera of additional video.
       * The rotation happens after the limit is exceeded, not proactively.
       * Moonfire NVR currently doesn't account for the unused space in the final
         filesystem block at the end of each file.
       * Moonfire NVR doesn't account for the space used for directory listings.
       * If a file is open when it is deleted (such as if a HTTP client is
         downloading it), it stays around until the file is closed. Moonfire NVR
         currently doesn't account for this.

When finished, start the daemon:

    $ sudo systemctl start moonfire-nvr

### System Service

Moonfire NVR can be run as a systemd service. If you used `prep.sh` this has
been done for you. If not, Create
`/etc/systemd/system/moonfire-nvr.service`:

    [Unit]
    Description=Moonfire NVR
    After=network-online.target

    [Service]
    ExecStart=/usr/local/bin/moonfire-nvr run \
        --db-dir=/var/lib/moonfire-nvr/db \
        --http-addr=0.0.0.0:8080
    Environment=TZ=:/etc/localtime
    Environment=MOONFIRE_FORMAT=google-systemd
    Environment=MOONFIRE_LOG=info
    Environment=RUST_BACKTRACE=1
    Type=simple
    User=moonfire-nvr
    Nice=-20
    Restart=on-abnormal
    CPUAccounting=true
    MemoryAccounting=true
    BlockIOAccounting=true

    [Install]
    WantedBy=multi-user.target

Note that the HTTP port currently has no authentication, encryption, or
logging; it should not be directly exposed to the Internet.

Complete the installation through `systemctl` commands:

    $ sudo systemctl daemon-reload
    $ sudo systemctl start moonfire-nvr
    $ sudo systemctl status moonfire-nvr
    $ sudo systemctl enable moonfire-nvr

See the [systemd](http://www.freedesktop.org/wiki/Software/systemd/)
documentation for more information. The [manual
pages](http://www.freedesktop.org/software/systemd/man/) for `systemd.service`
and `systemctl` may be of particular interest.
