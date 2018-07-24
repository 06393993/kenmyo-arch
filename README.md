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
  location: Location; 
}
```

* `base64` contains the content of the image in png format, encoded by base64
* `createdAt` contains the unix timestamp when the image is captured
* `createdBy` contains the device id which uploads this image
* `ownedBy` contains the user id who owns this image
* `location` contains the location information where the image is taken, created by the Camera

Where `Location` is defined as following:

```typescript
interface Location {
  desc: string;
  geolocation?: Atom<[ number, number ]>;
}
```

* `desc`: a short description of this location, for example: Kitchen, Living room, etc.
* `geolocation`: It's a 2-tuple including the latitude and langitude where the picure is taken. The first item of this tuple is latitude, and the second item is longitude. This property may be missing, since sometimes the Camera may fail to retrive the GPS information.

Now we can define the schema of the Image Storage.

```typescript
interface Images {
  [ id: number ]: Image;
  add: string => void;
}
```
Only get operation is supported. All set operation will be denied.

* Images will be indexed by ids.
* `add` function will add an image to the Image Storage. It accepts a png image encoded by base64 string as the only parameter. The server will automatically calculate the rest properties of the `Image` interface. Once an image is added, the Image Storage will emit an event to notify the system that a new image is added. If the device that calls the function is not owned by any of the user, the call should be denied.

S3, dynamodb and redis are all candidates for the permanent storage of Image Storage.

## Image Search

This service will pull images and the information on the detected objects in images from Object Detection and allow clients to search based the objects in the images. It stores the image as an Id, so it doesn't store the image itself. When a client searches for an object, this service will return an image id, a rectangle boundary where the object reside in this image and a confidence level of the detection. It should be sorted in descend order according to confidence level by default.

When the Object Detection finishes a job detecting objects on images, it will emit an event. And Image Search will listen to this event, and pull data accordingly.

The endpoint for the Image Search services is `/obj-to-img.json`. To introduce the data schema of Image Search, `DetectionResult` should be defined. It describes a rectangular area on one image and the confidence level.

```typescript
interface DetectionResult {
  image: number;
  object: string;
  area: [ [ number, number ], [ number, number] ];
  confidence: number;
}
```

* `image`: the Id fo the image where these detection results reside
* `object`: the object which these detection results are all about
* `area`: a rectagular area related to this detection result, the first item is the coordinate of the top left corner, and the second item is the coordinate of the bottom right
* `confidence`: a confidence level of this detection result, must between 0 and 1

Now we can define the data schema of ImageSearch.

```typescript
Interface ObjToImg {
  [ user: string ]: {
    [ object: string ]: {
      length: number;
      [ index: number ]: DetectionResult;
    };
  }
}
```
Only get operation is supported. All set operation will be denied.

All the DetectionResults are grouped and indexed by the users who owns the related image and the objects related to them. As mentioned, the order will be in the ascending order according to confidence level.

No external service is allowed to change the state of this service, so no function is available.

This service consists of a front router, a DetectionResults collector and an Elasticsearch service. Under the hood, it should be an internal Elasticsearch service that searches accorss all DetectionResult documents. However, it should only talk to the front router for adding DetectionResults and searching.

* front router is used to encapsulate the internal Elasticsearch service and exposed it as a Falcor DataSource, when a search request arrives, the front router is responsible for translating this request from Falcor json path to Elasticsearch search request and translate the result from Elasticsearch to a Falcor JSONGraph
* DetectionResults collector subscribes to Object Detection completion events. Once the Object Detection finishes the job on several images, the collector will pull data from Object Detection and push the data into the Elasticsearch service. This function is also responsible for notifying the Object Detection that the DetectionResults have been all archived in the Image Search service, so it's free for Objecat Detection to get rid of those results. It should be implemented as a AWS lambda function.
* Elasticsearch service is the core of the Image Search service. However, it should only talk to the front router and DetectionResults. Any external communication will not be allowed. 

## Object Detection

This service is responsible to generate the DetectionResults from images. When a Camera uploads an image to the Image Storage and the Image Storage emits an event, this service will respond to this event, and pull the image recently added. It will do the object detection job based every given time interval or there are too many images to process. Once a object detection job is done, it will store the results in the type of DetectionResult in a storage, and emit an event to notify the Image Search to pull the most recent results.

Obviously, this system is faced with several chanllenges.

* What if we don't have enough computation resources, should we just skip images or scale out our machine learning clusters?
* What if we are running out of space to store the DetectionResults, which results should be kept and which results should be deleted? What if the downstream Image Search service forgets to notify Object Detection that the detection results are archived?

These problems should be addressed in the future versions of this document, currently we just simply scale out the deep learning AMI instances and extend the capacity of the storage.

The end point of the Object Detection service is `/obj-detect-jobs.json`. The data schema is defined as following.

```typescript
interface DetectionJobs {
  jobs: {
    [ id: string ]: {
        length: number;
        [ index: number ]: Ref<DetectionResult>;
    };
  };
  results: {
    [ id: string ]: DetectionResult;
  };
  removeResults: (DetectionResult[]) => void;
}
```
Only get operation will be accepted. All set operation will be denied.

* `jobs`: jobs indexed by job id. Each job is an array of reference to DetectionResult. When the Object Detection emit an event indicating a job is done, the job id will be enclosed to that event.
* `results`: detection results indexed by id.
* `removeResults`: this function allow Image Search to inform Object Detection that the results have been archived, so the Object Detection may remove those results to free some space for the incoming results. This function will influence correspoding jobs as well.

Object Detection service consists of an image collector, a scheduler, a front router and a large machine learning cluster.

* image collector: the image collector is a lambda function to pull image data from Image Storage, once an image upload event emits and push the image id into a queue of unprocessed image queue.
* scheduler: the scheduler will frequently check the image queue, or be awaken when the queue contains too many images. It will fetch image details from Image Storage, dispatch images to machine learning nodes and pop the image id out of the unprocessed image queue. It may as well shut down or open machine learning nodes on demands. This can be done by using AWS Instance Scheduler. When the job is done, the node should send the results to the scheduler. It is also the scheduler's responsibility to find a place to store the results and scale the underlying storage instance on demand.
* front router: it encapsulate the inner storage as a Falcor DataSource.
* machine learning cluster: this cluster is under fully control of the scheduler, and only talk to the scheduler. It gets jobs from the scheduler and report the results to the scheduler. It can be shutdown or open according to the scheduler. The nodes in this cluster should be AWS EC2 GPU instances with AWS Deep Learning AMI on it. We will choose Tensorflow as the underlying calculation framework for now. Currently, they will use a object detection model directly from the Tensorflow detection model zoo. Dynamicly changing the model through HTTP request can be a future feature.

SQS or a queue inside the scheduler can be the candidates for the unprocessed image queue. Redis, dynamodb or S3 can be the candidates for the storage of detection results.

## Device Management

## Application Server

## Camera

## Frontend
