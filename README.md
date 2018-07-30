# Technical design document for Kenmyo

## Overview 
This project will use microservice architecture. There will be 5 different services. 

* Image storage 
* Image search 
* Object detection 
* Device management 
* Application Server 

And outside the system, there are two clients 

* Camera 
* Frontend 

The frontend is an abstract concept, it can be any agent that search stored pictures by words, for example, a web app, a mobile app or an Alexa skill. 
![](https://github.com/06393993/kenmyo-arch/blob/master/kenmyo-arch-0.0.png)
The dash line draws the boundary of Kenmyo backend system. Note that the authentication and the user system is not included in this graph, because they are light weight service directly provided by AWS, we will talk about the user system in device management and authentication in application server.

Here are some basic principals when implementing all microservices.

* Systems should communicate over HTTP, and should use Netflix Falcor technique (https://netflix.github.io/falcor/)
* All API endpoint should use lambda to implement
* Services may have their own permanent storage, but should not share between services
* Authentication for edge users(users and devices) should only happen at the edge of the system
* Authorization should only happen where the true access takes place
* Use AWS IAM to manage the authentication of internal services as much as possible
* Event driven

The following sections will introduce the 7 components one by one. For the 5 services, the data schema in typescript’s type notation will be included.

The events will not be included in this draft. The detailed design of events should be included in the next version of this documentation.

## Image storage

This is where images and related information stored. This service exposes following interfaces to the corresponding components:

* Add: allow Cameras to upload pictures
* Get by id: allow App Server and Object Detection to retrieve image and related information

### Data schema

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
  [ id: string ]: Image;
  add: (image: string, coord?: [number, number]) => void;
}
```
Only get operation is supported, and only the user who owns the picture can get the given images, except for the Image Search system, which has the full access to get any of the images. All set operation will be denied. The users' identity will be stored in the HTTP header.

* Images will be indexed by ids.
* `add` function will add an image to the Image Storage. This function can only be called by a Camera. The identity of the Camera will be stored in the HTTP header. It accepts a png image encoded by base64 string as the second parameter. The server will automatically calculate the rest properties of the `Image` interface. Once an image is added, the Image Storage will emit an event to notify the system that a new image is added. Only the Device Management can call this function. If the device that calls the function is not owned by any of the user, the call should be denied. How the camera should add an image will be included in the Camera section.

### Implementation

The whole system consists of a storage system including an S3 bucket and a dynamodb table to store the image itself and some meta data, a lambda to publish events on an AWS SNS topic when uploading to notify other services that an image has been uploaded, and a lambda served as the front router.

#### Storage

The storage includes a dynamodb table `image-storage.images` to store the meta data of images and an S3 bucket `image-storage.images` to store images themselves.

The data schema of the dynamodb table is:

```typescript
{
    id: string;
    s3: string;
    createdAt: number;
    createdBy: string;
    ownedBy: string;
    location: {
        desc: string;
        geolocation?: [ number, number ];
    }
}
```

* `s3`: the s3 url where the image stores
* `createdAt`: the unix timestamp when the image is uploaded

The `id` field should be the only primary key.

The meaning of other fields are the same as the `Image` schema. Only the Image Storage system components can visit these resources.

The S3 will store images in png encoded by base64.

#### Upload Event

The event will be published to a topic identified by `ImageCreatedTopicArn`. The schema of the message body of the event is:
```typescript
{
    id: string;
}
```
The id indicates the id of the image uploaded.

This event will be triggered by the dynamodb stream of `image-storage.images` table.

#### Front router

The front router resolves the incoming requests and manipulate the underlying system. According to the data schema, there are mainly three operations to implement:

* get `[*].<property>` where the `<property>` can be `base64`, `createdAt`, `createdBy`, `ownedBy`
* get `[*].location.<property>` where the `<property>` can be `desc` or `geolocation`
* call `add`

##### get `[*].<property>` and get `[*].location.<property>`

`createdAt`, `createdBy`, `ownedBy`, `location.<property>` can be directly retrieved from the dynamodb table `image-storage.images`.

For the `base64` field, the front router will first get the s3 url of the image by using the `s3` field from the table, and then retrieve the related image from S3, and fill the `base64` field by the returned value from S3 directly.

The router will check the if the incoming IAM role is attached to the `ImageStorageFullReadAccessPolicy` policy. If not, the router will deny the request when the `x-kenmyo-user-id` in the header isn't equal to the `ownedBy` field of the image requested. Otherwise, the image requested can always be retrieved.

##### call `add`

In order to insert an item into the `image-storage.images` table, all fields should be prepared. Here are how:

* `id`: generate a uuid for the image 
* `s3`: retrieve the base64 encoded image from the first paramter and then upload the content directly to the `image-storage.images` bucket and set this field by using the returned url.
* `createdAt`: use `+ new Date()` to get the current time
* `createdBy`: use the Camera id in the header `x-kenmyo-camera-id`
* `ownedBy` and `location.desc`: retrieve the Camera id from header `x-kenmyo-camera-id`, and query the `Device Management` through the path `devices.<camera-id>.["ownedBy", "locationDesc"]` to find the user who currently owns the Camera and the location description respectively. If no one owns the camera, the add function will not take effect.
* `location.geolocation`: if the second parameter `coord` is provided, use this as the value for the field

The router will check if the incoming IAM role is attached to the `ImageStorageCanAddPolicy` policy. If not, the call to `add` function will be denied.
This policy should only be attached to the Device Management service. Since the Image Storage will trust the Camera id provided in the request, so it's dangerous to assign other roles with this policy.

#### Parameters and Outputs

The whole system will be deployed as a CloudFormation stack, the parameters and outputs will be included.

##### Parameters

* `StackName`: the name of the stack to deploy to
* `ImageCreatedTopicArn`: the arn of the topic where the event to publish when an image is added to the dynamodb
* `ImagesS3Bucket`: the name of the S3 bucket where the images store, the stack will create the bucket
* `ImagesMetaTable`: the name of the dynamodb table where the metadata of images store, the stack will create the table
* `DeviceManagementAPIEndpoint`: the api endpoint of the Device Management service

##### Outputs

* `ImageStorageFullReadAccessPolicy`: the policy that allows a role to bypass the `x-kenmyo-user-id` authorization when request for images
* `ImageStorageCanAddPolicy`: the policy that allows a role to call `add` function
* `ImageStorageAPIEndpoint`: the url where the Image Storage service is hosted

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
Given a user, the user can only get the object that his id matches. All set operation will be denied. The users' identity will store in the header of the HTTP request.

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
Only get operation will be accepted. All set operation will be denied. Only the Image Search have access to this endpoint.

* `jobs`: jobs indexed by job id. Each job is an array of reference to DetectionResult. When the Object Detection emit an event indicating a job is done, the job id will be enclosed to that event.
* `results`: detection results indexed by id.
* `removeResults`: this function allows Image Search to inform Object Detection that the results have been archived, so the Object Detection may remove those results to free some space for the incoming results. This function will influence correspoding jobs as well.

Object Detection service consists of an image collector, a scheduler, a front router and a large machine learning cluster.

* image collector: the image collector is a lambda function to pull image data from Image Storage, once an image upload event emits and push the image id into a queue of unprocessed image queue.
* scheduler: the scheduler will frequently check the image queue, or be awaken when the queue contains too many images. It will fetch image details from Image Storage, dispatch images to machine learning nodes and pop the image id out of the unprocessed image queue. It may as well shut down or open machine learning nodes on demands. This can be done by using AWS Instance Scheduler. When the job is done, the node should send the results to the scheduler. It is also the scheduler's responsibility to find a place to store the results and scale the underlying storage instance on demand.
* front router: it encapsulate the inner storage as a Falcor DataSource.
* machine learning cluster: this cluster is under fully control of the scheduler, and only talk to the scheduler. It gets jobs from the scheduler and report the results to the scheduler. It can be shutdown or open according to the scheduler. The nodes in this cluster should be AWS EC2 GPU instances with AWS Deep Learning AMI on it. We will choose Tensorflow as the underlying calculation framework for now. Currently, they will use a object detection model directly from the Tensorflow detection model zoo. Dynamicly changing the model through HTTP request can be a future feature.

SQS or a queue inside the scheduler can be the candidates for the unprocessed image queue. Redis, dynamodb or S3 can be the candidates for the storage of detection results.

## Device Management

This service will allow users to manage their devices. Currently, users can link their accounts to Cameras from Cameras or unlink Cameras from either Cameras or a web management app. This service also stores the related information of the devices and provides interfaces to query and modify it.

The endpoint of this service is `/devices.json`. To define the data schema, we should first introduce the `Device` schema.

```typescript
interface Device {
  id: string;
  name: string;
  ownedBy?: string;
  locationDesc?: string;
}
```

* `id`: the id of the device. It is provided by AWS IoT SDK.
* `name`: the name of the device. It is editable by the user. This value should be kept in sync with the `thingId` of AWS IoT of this device.
* `ownedBy`: the id of the user who owns the device. If nobody owns the device, this field is missing.
* `locationDesc`: The location description that will enclosed to the image when the Camera uploads images. This field is optional.

Now we can define the data schema for Device Management.

```typescript
interface Devices {
  devices: { [id: string]: Device };
  devicesByUser: {
    [userId: string]: {
      length: number;
      [index: number]: Ref<Device>;
    }
  };
  unsetOwnership: () => void;

  setOwnership: () => void;
  unsetOwnershipFromDevice: () => void;
}
```
The get operation is supported to all the properties, but a user can only get the devices he/she owns. However only `devices[*].locationDesc` can be set, and only the device itself or the user owned the device are authorized to set the name of the device. Users' and Devices' identities will be stored in the HTTP header.

* `devices`: devices indexed by id.
* `devicesByUser`: devices array grouped by the id of users who own devices
* `unsetOwnership`: unlink the ownership relation between a device and a user. Only the user who owns the device will call this function with success

The following two functions can only be called by the device
* `setOwnership`: to call this function, the caller should be authenticated as both an IoT and a Cognito user. This function will change the ownership of the device no matter who owns it.
* `unsetOwnershipFromDevice`: call this function to remove any ownership relationship of this device 

Instead of directly talking to this service, the Camera should talk to this service through the AWS IoT service. The detailed will be included in the Camera section.

The underlying storage can be redis, dynamodb, or S3. However the attribute of the device should be stored in the attribute of the device in AWS IoT.

## Application Server and Frontend

The application server is a stateless server, and is the only server that frontends should talk to. It exposes all the functionality of the whole system to the external world, which currently includes: search an image on behalf of a user, list all the devices a user owns, remove the ownership of a device which is owned by a user, change the `locationDesc` of an owned device, get a specific image, get a specifc device.

```typesciprt
interface App {
  user: {
    [ email: string ]: {
      imagesByObjects: {
        [ object: string ]: {
          length: number;
          [ index: number ]: DetectionResult;
        }
      };
      devices: {
        length: number;
        [ index : number ]: Ref<Device>;
        remove: (id: string) => void;
      }
    };
  };
  devicesById: {
    [ id: string ]: Device;
  };
  imagesById: {
    [ id: string ]: Image;
  };
}
```
The previlege to get, set and remove to specific call is defined by the underlying system, the Application Server will not check.

* `user`: where all user-image and user-device relation is stored indexed by users' emails.
* `user[*].imageByObjects`: the interface where Image Search is exposed, detection results are grouped by the object in it. 
* `user[*].imageByObjects[*].length`: the length of the results array
* `user[*].imageByObjects[*][{integer}]`: one specific result
* `user[*].devices`: the interface where Device Manage is exposed
* `user[*].devices.length`: the length of the devices array
* `user[*].devices.remove`: the function to unlink the ownership relationship of a specific device of the given user
* `user[*].devices[{integer}]`: one specific device
* `devicesById[*]`: a specific device 
* `imagesById[*]`: a specific image

As an edge service, this service is also responsible for authentication. To let users have access to the funtionality above, the frontend should only call the interface provided by the Application Service. To login and sign up, use the Cognito service. The design of the web app frontend should not be included in this document.

## Camera

### Talk to the Kenmyo system

In order to communicate with different services. The Camera will directly talk to the AWS IoT service by using the Message Broke. The AWS IoT service will have Rules to call related lambda function on behalf of the Camera. The Camera can publish on following topics to implement different functionality:

* `/camera/{clientId}/uploadImage`: this topic is for uploading the images
* `/camera/{clientId}/setOwnership`: this topic is for changing the ownership of this device.
* `/camera/{clientId}/unsetOwnership`: this topic is for unlink the ownership relationship of this device

The schema of the payload of messages are:

#### `/camera/{clientId}/uploadImage`

```typescript
interface UploadImageMsg {
  image: string;
}
```

* `image`: the image to upload. Should be in the png format encoded in base64

#### `/camera/{clientId}/setOwnership`

```typescript
interface SetOwnershipMsg {
  userIdToken: string;
}
```

* `user`: the id token of the Cognito user

#### `/camera/{clientId}/unsetOwnership`

The payload of this message is empty.

### Initialization

During initialization, the AWS IoT should register the device, generate the cerficate, the related authorization rules. And the application runned on the Camera should download the certificate and related configuration files to its own storage to set up the running environment.

### Camera App

The app should host a local server.
The user should visit this page to set or unlink the ownership.
To set the ownership, the user should log in with his/her Cognito user.
After logging, the Camera owns the users' token.
The app should publish a message with the id token on the `/camera/{clientId}/setOwnership`.
On this page, the user can also unlink the relationship by publising a message on the `/camera/{clientId}/unsetOwnership`.
This page should also display the attribute of this camera elegantly.

In order to upload images periodically, the app should publish on the `/camera/{clientId}/uploadImage`.

### Attribute

Except the `id` and the `name`, the Camera should also include `locationDesc` and `ownedBy`. The attribute should be kept synced with the cloud by using AWS Device Shadow Service. However, the attribute should only be used to display, not for authentication and authorization.

## Authentication and Authorization

Authentication only happens at the edge of the whole system: AWS IoT and the Application Server.
The AWS IoT uses the X.509 to authenticate the incoming request from a device.
The Application Server use the id token from the Cognito to authenticate a user.
Once authentication completes, the internal system use `x-kenmyo-user-id` and `x-kenmyo-camera-id` to identify the user and the Camera respectively.

Authorization of the request from users and Cameras should happen in the microservice where the request is finnally handled.
Authorization of the request from internal services should use AWS AMI and AMI Policy to configure.
