+++
date = "2015-05-24T17:20:51-04:00"
draft = false
title = "MongoDB 3 on Debian Jessie"
tags = [ "mongodb", "debian", "jessie" ]
icon = "logo-mongo.svg"
+++

As of 2015-05-24, [the installation documentation](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-debian/) for MongoDB on Debian only included Wheezy (7). I found that it pretty much works fine on Jessie (8). But the installation as they had it had a couple problems:

* The `mongodb-org-3.0.list` file will point to a non-existent repository.
* The configuration file that ships with the Debian package is in the [old configuration format](http://docs.mongodb.org/v2.4/reference/configuration-options/), which makes it impossible to specify some of the options I needed.
* If you want to use the new [WiredTiger storage engine](http://www.wiredtiger.com/), the above issue combined with the fact that you've probably already started MongoDB makes this tricky.

## Install using the repository for wheezy

```bash
sudo apt-key adv --keyserver 'keyserver.ubuntu.com' --recv '7F0CEB10'
echo 'deb http://repo.mongodb.org/apt/debian wheezy/mongodb-org/3.0 main' | sudo tee '/etc/apt/sources.list.d/mongodb-org-3.0.list'
sudo apt-get update
sudo apt-get install -y mongodb-org
```

<div class="alert alert-warning">
<strong>Do not start MongoDB yet if you want to use WiredTiger!</strong>
</div>


## Replace the configuration

The new [YAML configuration format](http://docs.mongodb.org/manual/reference/configuration-options/) is way better than the old format, but the Debian wheezy package ships with the old format. So edit `/etc/mongod.conf` and replace it with equivalent YAML options. This is what I came up with. The `engine: "wiredTiger"` part is what prompted me to switch to the new configuration format.

```yaml
storage:
  dbPath: "/var/lib/mongodb"
  engine: "wiredTiger"
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy

systemLog:
  destination: file
  path: "/var/log/mongodb/mongodb.log"
  logAppend: true
  timeStampFormat: iso8601-utc

net:
  bindIp: "127.0.0.1"
  port: 27017
```

## WiredTiger

Hopefully you've specified in the configuration file that you want to use the wiredTiger storage engine *before* MongoDB starts for the first time. I did not do this. This leaves the `/var/lib/mongodb` directory full of files compatible only with MMAPv1. The easiest ay to fix this is just to delete all the files in the data directory after stopping MongoDB:

<div class="alert alert-warning">
<strong>Don't do this if you have data loaded already</strong>
</div>

```bash
sudo systemctl stop mongod.service
sudo rm -rf /var/lib/mongodb/*
```

Then add the `engine: "wiredTiger"` option, as shown above. Restarting should work fine. If you choose to invoke MongoDB without the `init` script / `systemctl` (which is useful with debugging), make sure that you use:

```bash
sudo -u mongodb mongod [... options]
```

Otherwise, `mongod` will create necessary files owned by either root or yourself, depending on whether you used `sudo` or not, which will lead to tricky issues such as the log (where error messages go!) not being writable by the `mongodb` user.
