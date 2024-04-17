# How to run MongoDB in a Docker container (running on an Ubuntu server) 

### Author: [Olav Tollefsen](https://www.linkedin.com/in/olavtollefsen/)

# Introduction

This article documents the process of how to setup and run MongoDB in a Docker container running on an Ubuntu server.

# What you need

You will one machine (physical or virtual) running Linux. We will be using Ubuntu Server 18.04.4 LTS. This article assumes that the Linux server have mounted one data disk (in addition to the operating system disk).

# Installing Ubuntu Server 18.04.4 LTS

Don't select to install Docker as a part of the operating system installation, since we want to have full control over the version we will install.

# Prepare the server

## Update operating system

After installing the operating system, make sure it's updated.

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

## Install Docker

### Set up the repository

Install packages to allow apt to use a repository over HTTPS:

```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

Add Docker’s official GPG key:

```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

```
$ sudo apt-key fingerprint 0EBFCD88
```

Set up the stable repository:

```
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### Install a specific version of Docker (19.03.8)

Update the apt package index

```
$ sudo apt-get update
```

 List the versions available in your repo:

```
$ apt-cache madison docker-ce
```

Install Docker version 19.03.8:

```
$ sudo apt-get install docker-ce=5:19.03.8~3-0~ubuntu-bionic docker-ce-cli=5:19.03.8~3-0~ubuntu-bionic containerd.io
```

To be able to issue Docker commands without sudo:
```
$ sudo usermod -aG docker <your-username>
```

Logout from the current user and login again for the above command to take effect.

# Prepare the filesystem for the MongoDB data files

List the disks on the server:

```
$ sudo lshw -class disk -short (or lsblk -o KNAME,TYPE,SIZE,MODEL)
```

The output should look something like this:
```
H/W path      Device       Class      Description
=================================================
/0/1/0.0.0    /dev/cdrom   disk       Virtual CD/ROM
/0/2/0.0.0    /dev/sda     disk       136GB Virtual Disk
/0/3/0.0.0    /dev/sdb     disk       136GB Virtual Disk
```

In this case the /dev/sda is refering to the operating system disk and /dev/sdb is the data disk.

Warning! Make sure to understand what disks you have mounted to make sure you are not deleting data unintentionally.

## Format the data disk

This assumes that the data disk is blank.

Create a new partition by issuing the following command:
```
$ sudo fdisk /dev/sdb
```

Enter 'n' at the fdisk command prompt to add a new partition. Select defaults for all the inputs and remember to select "w" when returning to the fdisk command prompt in order to write table to disk and exit.

Format the partition as XFS:

```
sudo mkfs.xfs -L mongodb /dev/sdb1
```

## Mount the new filesystem

Create the mountpoint

```
$ sudo mkdir /mnt/mongodb
```

Mount the new filesystem

```
$ sudo mount -t xfs /dev/sdb1 /mnt/mongodb
```

## Make sure new filesystem is mounted at boot

Get UUID for newly created file system

```
$ sudo blkid /dev/sdb1
```

Add this to "/etc/fstab":

```
UUID=<UUID> /mnt/mongodb xfs defaults 1 1
```

# Run MongoDB in a Docker container

```
$ docker run --name mongodb -p 27017:27017 -v /mnt/mongodb:/data/db -d --restart=always mongo
```

# Connect to MongoDB in container

```
docker exec -it mongodb bash
```

Now you can start the mongo shell by issuing the command:

```
mongo
```

## Some useful MongoDBH commands

### List databases

```
> show dbs
```

### Set curent database

```
> use <database-name>
```

### Display curent database

```
> db
```

### Delete current database

```
> db.dropDatabase()
```

### Create a new database

If a database does not exist, MongoDB creates the database when you first store data for that database. As such, you can switch to a non-existent database and perform the following operation in the mongo shell:

```
> use <new-database-name>

> db.myNewCollection1.insertOne( { x: 1 } )
```

### Create a new collection (explicitly)

```
> db.createCollection(<name>)
```

### Indexes

Create simple indexes:
```
> db.imageProperties.createIndex( { "imagePath": 1 } )
```

```
 > db.imageProperties.createIndex( { "imageProperties.dateTimeOriginal": -1 } )
```

Create text index:
```
> db.imageProperties.createIndex(
  {
    "imageProperties.cameraMaker": "text",
    "imageProperties.cameraModel": "text",
    "imageProperties.lensMake": "text",
    "imageProperties.lensModel": "text",
    "imageAnalysis.categories.name": "text",
    "imageAnalysis.color.dominantColors": "text",
    "imageAnalysis.description.tags": "text",
    "imageAnalysis.description.captions.text": "text",
    "imageAnalysis.faces.gender": "text",
    "imageAnalysis.tags.name": "text",
    "mapProperties.addresses.address.street": "text",
    "mapProperties.addresses.address.streetName": "text",
    "mapProperties.addresses.address.countrySubdivision": "text",
    "mapProperties.addresses.address.municipality": "text",
    "mapProperties.addresses.address.country": "text",
    "mapProperties.addresses.address.countryCode": "text",
    "mapProperties.addresses.address.countryCodeISO3": "text",
    "imageProperties.dateTimeOriginal": -1,
  },
  { name: "TextIndex" }
)
```

List indexes:

> db.imageProperties.getIndexes()

Delete index:
```
> db.imageProperties.dropIndex("TextIndex")
```

Take a look at index usage:
```
> db.imageProperties.aggregate( [ { $indexStats: { } } ] )
```

## Example queries

Simple find
```
> db.imageProperties.find({ "imagePath" : "2020/2020-01-04/IMG_0931.JPG" })
```

Return count of documents
```
> db.imageProperties.count()
```

Return distinct values:
```
> db.imageProperties.distinct("imageProperties.cameraMaker")

> db.imageProperties.distinct("imageProperties.cameraModel")
```

Return all docuements after a date / time:
```
> db.imageProperties.find({ "imageProperties.dateTimeOriginal" : { $gte: "2018-01-01T00:00:00.0000000" }})
```

Return only the "imageProperties.dateTimeOriginal" field:
```
> db.imageProperties.find({ "imagePath" : "2017/2017-01-01/IMG_0706.JPG" }, {"imageProperties.dateTimeOriginal" : 1, _id: 0})
```

Return matching documents between two dates:
```
> db.imageProperties.find( { $and: [ { "imageProperties.dateTimeOriginal": { $gte: "2017-01-01T00:00:00.0000000" } }, { "imageProperties.dateTimeOriginal": { $lte: "2017-02-01T00:00:00.0000000" } } ] } )
```

Return matching documents between two dates and return the "imageAnalysis.tags.name" field:
```
db.imageProperties.find( { $and: [ { "imageProperties.dateTimeOriginal": { $gte: "2020-10-28T00:00:00.0000000" } }, { "imageProperties.dateTimeOriginal": { $lte: "2020-10-29T00:00:00.0000000" } } ] }, {"imageAnalysis.tags.name" : 1, _id: 0} )
```

Search text:
```
> db.imageProperties.find( { $text: { $search: "car" } }, {"imagePath" : 1, _id: 0 } )
```

## Backup the database to a file

```
 $ /usr/bin/mongodump --forceTableScan --uri=mongodb://mongodb:27017/olavt-images --archive > mongodb.dump
 ```