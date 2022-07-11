# ISSIA22: Node-RED Software Agent Workshop
Workshop on how to develop software agents using Node-RED and semantic web technologies held during the 2022 International Summer School on Industrial Agents

## Folder structure:
- `flows`: Final flows that can be imported and tested in Node-RED.
- `templates`: Unfinished flows that may be used to follow along during or after the workshop
- `ontology`: OWL ontology used to describe agents and their capabilities. Checkout https://github.com/hsu-aut/Industrial-Standard-Ontology-Design-Patterns for more ontologies

## Use Case:

<img src="https://github.com/hsu-aut/ISSIA22-node-red-agents/blob/images/images/UseCase.png?raw=true" align="left" width="400px"/>
In this workshop, we look at a use case of flexible production. A conveyor is used to transport workpieces from the left (Sensor A) to the right (Sensor B). Sensor A is able to detect workpieces and can measure the time it takes a workpiece to pass through. Together with the conveyor speed, the workpiece length can be calculated. As soon as a workpiece arrives at Sensor B, B needs to get the length from A in order to place the workpiece centrically. Then, a gripper is selected depending on the workpiece length. While Gripper A is able to lift larger (i.e. longer) workpieces, it is also more expensive to operate. Gripper B can only handle shorter workpiece but has lower operation costs. 

<br clear="left"/>

The use case is implemented in two different ways. We start with a more simple approach in which data is stored locally (i.e. in a Node-RED flow) and agents are "hard-wired" together.
After this initial approach, we look at a way to increase flexibility by using semantic web technologies such as an ontology and SPARQL queries. Agents can be registered and deregistered. Instead of being hard-wired, selecting agents is more dynamic and based on the currently registered agents.

### Approach 1: Local data storage and direct communication
The flow of Sensor A starts with an inject node that is used to simulate an arriving workpiece. The inject node creates an empty payload as well as a random msg.delay. In the subsequent function node, an ID is generated (= simulating detection) and the start time is set. The delay node takes the msg.delay that was in the inject node and pauses the execution for the given delay value. This simulates the workpiece passing through the sensor.

Then, the time difference between start and end of detection is calculated and used to request the length from the conveyor. We assume that only the conveor knows its speed and thus Sensor A cannot calculate the workpiece length by itself. Sensor A uses an HTTP request node to request the workpiece length from the conveyor. After getting a response, Sensor A stores the workpiece length locally (i.e. in the flow using `flow.set()`).

![image](https://user-images.githubusercontent.com/50097079/178309446-c1343b92-611f-494e-a6c5-cc24c856ffe6.png)

The inject node of Sensor B is used to simulate arrival of the workpiece at B. The last workpiece ID is looked up (it is stored globally to simulate workpiece order). Sensor B requests the length of the workpiece with the given ID from Sensor A. After getting the length, B calculates the time it takes to stop a workpiece at half its length and requests the conveyor to stop. As soon as the conveyor stops, Sensor B selects a gripper using a simple if-condition. The selected Gripper is then requested to handle the workpiece. 

![image](https://user-images.githubusercontent.com/50097079/178311903-6ddd3ee2-ab97-4e3a-ba80-811abe77534e.png)


### Approach 2: Using an ontology as a semantic information mode / data store

