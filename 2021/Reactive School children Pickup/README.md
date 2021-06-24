# Use case

## Background

There is a service called round-about at my children's school, where you can pick up the children from school in your car.
The idea is quite nice, although because of the location of the school and the way it is run it is not very efficient and creates quite a traffic jam during pick-up time.
The low efficiency is due to the way they intercept an incoming parent's car and match it to the right children. 
There is a network of walkie-talkies and teachers interacting with each other to create that match, and sometimes they need to check again if that vehicle is the right one. If you are lucky and for some reason, your child is already there waiting, it is a matter of seconds; on the other hand, in the worst case, you have to wait up to 10 minutes.

If we compare it to the computer world, this is a classic integration of request and response where the teachers are the service, and the walkie-talkie is the protocol used to communicate.

## Proposal

The proposal is to improve this process with better integration by leveraging Event-Driven Architecture (EDA) and AI/ML to recognise parents and children (face detection) and number plates using OCR (Optical Character Recognition).

In this way, when a car approaches the school, Machine Learning running in a single OpenShift Edge Node detects the number plate and targets the right children. The ML has to match the number plate, but at the same time, for the safety of the children, it has to check if those driving the car are the parents using facial recognition. 
In case of a positive match, the system will alert the child. Each child will have an IoT device that receives the notification that the parents are approaching, and the child has to go to the round-about. The child should acknowledge the message, and in case of delay, it would be possible to track the child and alert the primary teacher to check.
Of course, the system has to allow for a few exceptions; for example, sometimes parents pick up other children. Thanks to the administration panel, it is possible to register parent B, who will pick up the kids of parent A. Then the AI will check them at the entrance.

With this responsive and autonomous approach, the queue time will decrease dramatically, and the round-about system's efficiency will improve.

The individual node will be part of the school's mesh network.

# Technology stack (In WIP)

## Hardware Stack

* Sigle Edge Node for AI/ML
* Central OpenShift Cluster
* Cameras for face and plate detection.
* IoT devices for children

## Software Stack

* Quarkus