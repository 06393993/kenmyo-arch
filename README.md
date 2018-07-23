# Technical design document for Kenmyo

## Overview 
This project will use microservice architecture. There will be 5 different services. 

*	Image storage 
*	Image search 
*	Object detection 
*	Device management 
*	Application Server 

And outside the system, there are two clients 

*	Camera 
*	Frontend 

The frontend is an abstract concept, it can be any agent that search stored pictures by words, for example, a web app, a mobile app or an Alexa skill. 
![](https://github.com/06393993/kenmyo-arch/blob/master/kenmyo-arch-0.0.png)
The dash line draws the boundary of Kenmyo backend system. Note that the authentication and the user system is not included in this graph, because they are light weight service directly provided by AWS, we will talk about the user system in device management and authentication in application server.

Here are some basic principals when implementing all microservices.

*	Systems should communicate over HTTP, and should use Netflix Falcor technique (https://netflix.github.io/falcor/)
*	All API endpoint should use lambda to implement
*	Services may have their own permanent storage, but should not share between services
*	Authentication and authorization should only happen at the edge of the system
*	Event driven

The following sections will introduce the 7 components one by one. For the 5 services, the data schema in typescript’s type notation will be included.

The events will not be included in this draft. The detailed design of events should be included in the next version of this documentation.

## Image storage

This is where images and related information stored. This service exposes following interfaces to the corresponding components:

* Add: allow Cameras to upload pictures
*	Get by id: allow App Server and Object Detection to retrieve image and related information

The endpoint of the Image Storage is `/images.json`. In order to define the data schema of Image Storage, we should first define `Image` interface.

```typescript
interface Image {
  base64: string;
  createdAt: number;
  createdBy: string;
  ownedBy: string;
}
```

* `base64` contains the content of the image in png format, encoded by base64
* `createdAt` contains the unix timestamp when the image is captured
* `createdBy` contains the device id which uploads this image
* `ownedBy` contains the user id who owns this image

Now we can define the schema of the Image Storage.

```typescript
interface Images {
  [ id: number ]: Image;
  add: string => void;
}
```

* Images will be indexed by ids.
* `add` function will add an image to the Image Storage. It accepts a png image encoded by base64 string as the only parameter. The server will automatically calculate the rest properties of the `Image` interface. Once an image is added, the Image Storage will emit an event to notify the system that a new image is added.

S3, dynamodb and redis are all candidates for the permanent storage of Image Storage.

## Image Search

## Object Detection

## Device Management

## Application Server
