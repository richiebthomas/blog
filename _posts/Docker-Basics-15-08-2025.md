---
layout: post
title: "Running MongoDB and Mongo Express on Docker"
date: 15th Aug 2025
tags: Docker, MongoDB, MongoExpress, Troubleshooting
excerpt_separator: <!--more-->
---

When I'm learning Docker, one of the best starter projects is to run a database and a web UI together. In this tutorial, I'll set up **MongoDB** and **Mongo Express** so I can manage my database right from my browser.<!--more-->  
But here's the twist: I'll also walk through a *very common mistake* — naming mismatches — that prevents Mongo Express from connecting, and how to fix it for good.

---

## Why MongoDB + Mongo Express?
MongoDB is a popular NoSQL database.  
Mongo Express is a lightweight web-based admin panel for MongoDB, making it easy to:
- Browse collections
- Add, update, and delete documents
- Run queries without a CLI

Running them together in Docker is a perfect way to understand container networking.

---

## The Goal
I want to:
1. Run MongoDB with authentication.
2. Run Mongo Express and connect it to MongoDB.
3. Access Mongo Express in the browser.
4. Avoid (and fix) the common *container name mismatch* error.

---

## Step 1 — Create a Docker Network
Containers on the same network can talk to each other using container names as hostnames.

```bash
docker network create mongo-network
````

---

## Step 2 — The **Wrong** MongoDB Setup (The Mistake I Made First)

When I first ran MongoDB, I used:

```bash
docker run -d -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  --name mongodb \
  --net mongo-network \
  mongo
```

Looks fine, right?
But then I ran Mongo Express like this:

```bash
docker run -d -p 8081:8081 \
  -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
  -e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
  -e ME_CONFIG_MONGODB_SERVER=mongo \
  --name mongo-express \
  --net mongo-network \
  mongo-express
```

Mongo Express tries to connect to `mongo:27017` — but my database container is named `mongodb`, so Docker DNS couldn't resolve it.

---

## Step 3 — The Error I Saw

The Mongo Express logs showed:
![Mongo Express Error Screenshot](https://raw.githubusercontent.com/richiebthomas/blog/refs/heads/main/assets/images/Docker-Basics-15-08-2025/Screenshot%202025-08-15%20151626.png "Mongo Express Connection Error")
*The Mongo Express container logs after attempting to connect to a wrong container*

In plain English:

> "I can't find any container called `mongo` on this network."

---

## Step 4 — The Fix: Match the Names

I had two choices:

* Change `ME_CONFIG_MONGODB_SERVER` to `mongodb`
* Or rename the database container to `mongo`

I chose the second option because `mongo` is shorter and cleaner.

```bash
docker rm -f mongodb

docker run -d -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  --name mongo \
  --net mongo-network \
  mongo
```

Now `--name mongo` matches what Mongo Express is expecting.

---

## Step 5 — Run Mongo Express (Corrected)

```bash
docker rm -f mongo-express

docker run -d -p 8081:8081 \
  -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
  -e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
  -e ME_CONFIG_MONGODB_SERVER=mongo \
  --name mongo-express \
  --net mongo-network \
  mongo-express
```

---

## Step 6 — Test It

* Visit: [http://localhost:8081](http://localhost:8081)
* Log in with:

  * Username: `admin`
  * Password: `pass` (default, change it later)

I should now see the Mongo Express UI without any connection errors.

---

## Lessons Learned

* **Docker DNS uses container names** on a user-defined network.
* Always make sure `ME_CONFIG_MONGODB_SERVER` matches the database container name exactly.
* Use `docker logs <container>` as my first troubleshooting tool.
* If something's misconfigured, `docker rm -f` and re-run with the correct settings — it's faster than debugging broken containers.

---

## Final One-Liners

Here's the working setup in two commands:

```bash
docker run -d -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=password \
  --name mongo \
  --net mongo-network \
  mongo
```

```bash
docker run -d -p 8081:8081 \
  -e ME_CONFIG_MONGODB_ADMINUSERNAME=admin \
  -e ME_CONFIG_MONGODB_ADMINPASSWORD=password \
  -e ME_CONFIG_MONGODB_SERVER=mongo \
  --name mongo-express \
  --net mongo-network \
  mongo-express
```

---

![Mongo Express Dashboard](https://raw.githubusercontent.com/richiebthomas/blog/refs/heads/main/assets/images/Docker-Basics-15-08-2025/Screenshot%202025-08-15%20150510.png "Mongo Express Dashboard")
*The Mongo Express dashboard after a successful connection*

```
