---
title: "Integration testing with MongoDB on Gitlab-CI"
description: "How to do integration tests with a MongoDB database as a service on Gitlab-CI."
# 1. To ensure Netlify triggers a build on our exampleSite instance, we need to change a file in the exampleSite directory.
theme_version: '2.8.2'
date: 2020-11-19
publishDate: 2020-11-19
cascade:
  featured_image: '/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png'
---

At Slickteam our CI-CD platform works with Gitlab-CI, and works well! We are grasping more and more capabilities of the CI project after project, and we want to do things in the best possible way. For one of my project I had written integration tests with the database, and I wanted them to be run automatically on the CI pipeline.

To be able to do this, I have read and tested many things. For my test I need to have a database initialized with empty collections, and a user with read-write access to the database.

First I wanted to have it directly on the pipeline without relying on another Docker image than the MongoDB official. But after many tries I didn’t find a way to have it working as I wanted, and the pipeline stage contained too many commands to be readable and maintainable correctly.

After that I decided to have my MongoDB works like a service in Gitlab-CI, and for that to work well I must have a Docker image with a MongoDB service and a database configured like I want.

First of all, I must have a Docker image with a pre-configured MongoDB inside. To do that, I have created a project dedicated to this.

This project contains 3 files: the configuration of Gitlab-CI, a Dockerfile and a file containing the initialization script for MongoDB.

You can initialize a MongoDB database that is running inside a container through a javascript file. This file will contain mongo shell instructions to set up your database. I use it to create a database user, set his rights and create collections. It looks like that:
Init script for the database

With this script done, I have a Dockerfile to build my image with this script and set the root user for MongoDB:
Dockerfile for testing image

To complete this, I have my .gitlab-ci.yml file so the image will be build and push to our repository automatically:
.gitlab-ci file for testing image

Now that we have our configured MongoDB image in our repository, we can use it in a project to run integration tests with it.

In Gitlab-CI, we have a complete pipeline to build our project. One stage runs the integration tests, and for that we will use our new Docker image to have a MongoDB service configured as we want.

Next is an example of an integration test class we have, our project is written in Kotlin (with Javalin) and we use KMongo to connect to our database:
Integration test class

I have also configured the stage so the results of the tests will be displayed inside Gitlab-CI pipeline run page.
Stage in gitlab-ci.yml for integration tests

In this stage I have set an alias for our service (mongo) and I use it in the connection URI to MongoDB.

Now if you run your pipeline your tests must be working:
![Tests results in Gilab-CI UI](/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green.png "Tests results in Gilab-CI UI")

The alias for the service is not known locally, so we have a little modification to do in our configuration so we can run the tests locally also.

To have our integration tests running locally, I have started by writing a docker-compose file so I can have a MongoDB:
docker-compose file to run locally MongoDB

Now we need to have a configuration that will set the mongo host as localhost and not mongo like on the CI. To have 2 different configurations I have set a priority order in the configuration of the library I use (Konfig):
Priority in configurations

So now if I have a dev.properties file in the local directory of the execution of my application it will override the default file. I added this file in the root of my project, and to ensure that it will be not commited I added it to the .gitignore file. In my dev.properties file I put the same properties as in the default one so I can manage easily different configuration if necessary:
MongoDB connection configuration

And that’s it! Now the tests can be run in your IDE:
![It’s green !](/images/int_testing_mongo_gitlab/int_testing_mongo_gitlab_all_green_ide.png "It’s green !")

You can find all the file and a working project on this github repo. I hope this will help you to test your application.

If you liked this article or you found it useful, don’t hesitate to share it ! 🙂
