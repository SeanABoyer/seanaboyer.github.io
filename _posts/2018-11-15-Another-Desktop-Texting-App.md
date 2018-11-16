---
layout: post
title: Another Desktop Texting Application with a private database repository 
---
How I began learning angularJS by attempting to recreate some well known desktop texting applications like pushbullet, yappy, etc.

## Problem
I had recently changed jobs when I started this project and we were planning on moving to a portal that was written on the angualrJS framework and nobody on the team had any expierence with angularJS. I had some expierence at my past job tinkering with the portal and how it implemented angularJS (a lot is obfuscated), but I never developed an applicaiton from scratch using angualrJS. I had used many applications to text from my desktop and was curious how the inner workings worked, so I decided I would try and create a 


## Quick Solution OverView
I created a REST API for a database with a simple schema utilzing [Flask-SQLAlchemy](flask-sqlalchemy.pocoo.org/). I then used a python library to implement [mqtt](https://mqtt.org/) to allow for devices (Moblie and Web) to subscribe to changes in the database. On the same device that hosted the database I created a simple web application written in angualrJS to subscribe to the mqtt service for incomming messages (from the mobile app) and POST to the database for outgoing messages (to the mobile app). Finally, I created a native Android mobile app (that I no longer have the code for due to formatting the harddrive that this code was on) that required the IP address of the device that was hosting the database (in my case a raspberryPI) and a username and password. The mobile app subscribed to the device (raspberryPi) upon logging in for incomming messages (from the web app) and submitted to the database for outgoing messages (when sent from the phone rather than the web app).

This was a really fun side project to play with that allowed me to learn about MQTT and angualrJS, also gave me a great reason to bring out the raspberry pi to play with.

## In-Depth Solution OverView
I started off with the database schema which ended up being fairly simple. I wanted to be able to support multiple users using one database, supporting multiple conversations.
### The Database Schema
```
Converation:
    PhoneNumber (Whom you were texting)
    User (The originating User)

Message:
    To PhoneNumber
    From PhoneNumber
    Message
    Time
    Converation

User:
    UserName
    Password
    CreationTime
    PhoneNumber
```

All of the code for the models can be found under the [Models Folder](https://github.com/SeanABoyer/Home_Texting_DB-API/tree/master/Models).

### The REST API 
After getting the database established I wrote up the REST API using Flask to handle a lot of the setup of resources. I mainly consumed the data via simple endpoints, did some validation on the user entered data and then saved it into the database. On saving of a message to the database, I would build an object to publish to the mqtt message bus that contained the message ID, that way the web app and mobile app could connect to the message bus and query back into the database to obtain the message.

All of the code for the rest api can be found under the [Controllers Folder](https://github.com/SeanABoyer/Home_Texting_DB-API/tree/master/Controllers) and [Other Folder](https://github.com/SeanABoyer/Home_Texting_DB-API/tree/master/Other)

### The Mobile App
_I did lose all my code for the mobile application due to formatting my hard drive and not moving the code off in time(Hmm... I swear there is a really cool website that lets you store code in a repository for free, if only I was using it...)._

Essentially the mobile app came up with a screen that asked for the IP address of the device hosting the database. After submitting the IP addreess, it would check to make sure the device was pingable and then the user would be navigated to another screen that requested a username and password. After supplying the username and password it would subscribe to the mqtt message bus and any incoming messages to the phone would be sent to the database, while any messages published on the message bus where the source was not the mobile app would be sent to the phone number supplied. I did run into an issue where messages could get caught in a loop of sorts and could spam a phone number (sorry to my wife and best friend), I ended up putting in a workaround to check if the previous message sent was equal to the current message that was trying to be sent. This worked a majority of the time, but I knew it was not a true solution.

### The Front End
This was my very first angularJS application as well as my first time using a web framework. I did not want to get too deep into the design because I figuered if I wrote the angualr correctly I could change the design of the UI and bind the functions to new elements where necessary. I started with just trying to get messages from the database to display in the conversations area. Soon after succeeding with that I worked on trying to keep it up to date with inserts into the database, I utilized a MQTT library to listen for new publishes and do an API call to the database on inserts. After I got that working I wrapped things up with being able to send new messages to the database that my mobile app would pick up and send from there. 

All of the code for the front end can be found under the [FrontEnd Folder](https://github.com/SeanABoyer/Home_Texting_DB-API/tree/master/FrontEnd).