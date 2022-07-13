# ISSIA22: Node-RED Software Agent Workshop
Workshop on how to develop software agents using Node-RED and semantic web technologies held during the 2022 International Summer School on Industrial Agents

## Table of Contents
1. [Installation instructions](https://github.com/hsu-aut/ISSIA22-node-red-agents#installation-instructions-what-you-need)
2. [Folder structure](https://github.com/hsu-aut/ISSIA22-node-red-agents#folder-structure)
3. [Use Case](https://github.com/hsu-aut/ISSIA22-node-red-agents#use-case)
4. [Approach 1 - Local data store, hard-wired communication](https://github.com/hsu-aut/ISSIA22-node-red-agents#approach-1-local-data-storage-and-direct-communication)
5. [Approach 2 - Ontology and agent communication](https://github.com/hsu-aut/ISSIA22-node-red-agents#approach-2-using-an-ontology-and-more-detailed-agent-communication)
    1. [Registering & Unregistering Agents](https://github.com/hsu-aut/ISSIA22-node-red-agents#registering-and-unregistering-agents-with-an-explicit-capability-description)
    2. [Sensor A](https://github.com/hsu-aut/ISSIA22-node-red-agents#sensor-a---detect-and-store-workpiece-information-in-graphdb)
    3. [Conveyor](https://github.com/hsu-aut/ISSIA22-node-red-agents#sensor-a---detect-and-store-workpiece-information-in-graphdb)
    4. [Sensor B - React on Products and Create CFP](https://github.com/hsu-aut/ISSIA22-node-red-agents#sensor-b---wait-for-products-get-relevant-mqtt-topic-and-create-proposals)
    5. [Grippers - Handle CFPs](https://github.com/hsu-aut/ISSIA22-node-red-agents#grippers---reacting-on-cfps)
    6. [Sensor B - Wait for proposals](https://github.com/hsu-aut/ISSIA22-node-red-agents#sensor-b---waiting-for-proposals)
    7. [Grippers - React on Accept / Reject](https://github.com/hsu-aut/ISSIA22-node-red-agents#grippers---reacting-on-accept--reject)
6. [Limitations](https://github.com/hsu-aut/ISSIA22-node-red-agents#limitations)


## Installation instructions: What you need
- Node-RED: Install Node-RED (requires Node.js and npm) as described in the [Node-RED installation tutorial](https://nodered.org/docs/getting-started/local)
- GraphDB: Get a free version of it at https://www.ontotext.com/products/graphdb/graphdb-free/. Starting GraphDB should open your browser at localhost:7200, where you will find GraphDB's user interface. Important: Make sure to create a repository named "AgentSummerSchool" as this repository is used by all the nodes in Node-RED sending queries. You could also chose a different repository name, but then you have to make sure to change all paths of nodes interacting with GraphDB in Node-RED. You also have to import the ontology (without individuals) into this repository.

## Folder structure:
- `flows`: Final flows that can be imported and tested in Node-RED. Note that there is a subfolder `local` with purely local implementations of Sensor A, Sensor B and the two grippers.
- `templates`: Unfinished flows that may be used to follow along during or after the workshop
- `ontology`: OWL ontology used to describe agents and their capabilities. Checkout https://github.com/hsu-aut/Industrial-Standard-Ontology-Design-Patterns for more ontologies

## Use Case:

<img src="https://github.com/hsu-aut/ISSIA22-node-red-agents/blob/images/images/UseCase.png?raw=true" align="left" width="400px"/>
In this workshop, we look at a use case of flexible production. A conveyor is used to transport workpieces from the left (Sensor A) to the right (Sensor B). Sensor A is able to detect workpieces and can measure the time it takes a workpiece to pass through. Together with the conveyor speed, the workpiece length can be calculated. As soon as a workpiece arrives at Sensor B, B needs to get the length from A in order to place the workpiece centrically. Then, a gripper is selected depending on the workpiece length. While Gripper A is able to lift larger (i.e. longer) workpieces, it is also more expensive to operate. Gripper B can only handle shorter workpiece but has lower operation costs. 

<br clear="left"/>

The use case is implemented in two different ways. We start with a more simple approach in which data is stored locally (i.e. in a Node-RED flow) and agents are "hard-wired" together.
After this initial approach, we look at a way to increase flexibility by using semantic web technologies such as an ontology and SPARQL queries. Agents can be registered and deregistered. Instead of being hard-wired, selecting agents is more dynamic and based on the currently registered agents.

## Approach 1: Local data storage and direct communication
The flow of Sensor A starts with an inject node that is used to simulate an arriving workpiece. The inject node creates an empty payload as well as a random msg.delay. In the subsequent function node, an ID is generated (= simulating detection) and the start time is set. The delay node takes the msg.delay that was in the inject node and pauses the execution for the given delay value. This simulates the workpiece passing through the sensor.

Then, the time difference between start and end of detection is calculated and used to request the length from the conveyor. We assume that only the conveor knows its speed and thus Sensor A cannot calculate the workpiece length by itself. Sensor A uses an HTTP request node to request the workpiece length from the conveyor. After getting a response, Sensor A stores the workpiece length locally (i.e. in the flow using `flow.set()`).

![image](https://user-images.githubusercontent.com/50097079/178309446-c1343b92-611f-494e-a6c5-cc24c856ffe6.png)

The inject node of Sensor B is used to simulate arrival of the workpiece at B. The last workpiece ID is looked up (it is stored globally to simulate workpiece order). Sensor B requests the length of the workpiece with the given ID from Sensor A. After getting the length, B calculates the time it takes to stop a workpiece at half its length and requests the conveyor to stop. As soon as the conveyor stops, Sensor B selects a gripper using a simple if-condition. The selected Gripper is then requested to handle the workpiece. 

![image](https://user-images.githubusercontent.com/50097079/178311903-6ddd3ee2-ab97-4e3a-ba80-811abe77534e.png)

Please note that this approach was just discussed, but not fully implemented during the workshop. You can try it out by importing the flows inside the `flows\local` folder. Make sure to also import the conveyor.

## Approach 2: Using an ontology and more detailed agent communication

With this second approach, we want to increase flexibility of the overall system and we want to use an explicit model of agents, their capabilities as well as constraints on how & when capabilities may be used. Furthermore, we also want to add a more "realistic" kind of agent communication. MQTT is used to dynamically assign groups of agents and communicate with them without the need to create a direct connection.
For this approach to work, you have to set up a Graph DB instance on your machine. This instance has to have a repository with name + ID = "AgentSummerSchool" into which the ontology `AgentSummerSchool-Ontology_merged.owl` has to be imported. 

### Registering and unregistering agents with an explicit capability description
<img src="https://user-images.githubusercontent.com/50097079/178440843-f6caf33f-6f1f-424e-b3fd-e9c6b4ab77da.png" align="left" width="400px"/>

We start by providing a way for agents to register themselves and their capabilities by clicking on an inject node. To achieve that, a Directory Facilitator (DF) agent is added, which acts like a registry. 

It allows agents to register themselves and manages MQTT topics. Agents can send a SPARQL INSERT query in their msg.payload to the DF via HTTP. The DF will forward the query to the GraphDB effectively registering the agent. It then takes the msg.payload.sender to retrieve information about the agent it just registered. The capabilities of the agent are taken and used as MQTT topics. The DF sets the topic by sending an HTTP request back to the agent.

![image](https://user-images.githubusercontent.com/50097079/178442301-401ceb71-cb50-4c8f-81a3-25631e841b2d.png)

<img src="https://user-images.githubusercontent.com/50097079/178442839-f663c0a4-c3bf-4bde-bf35-9ef1a51173d6.png" align="left" width="400px"/>
In order for the DF to be able to pass the MQTT topic back to an agent, agents (like Gripper A and B) provide an "http in" node. The url is contained in the agent description (i.e. the INSERT query) and can thus be dynamically found by the DF. In this way, both Gripper A and Gripper B can be registered. Deregistering works just similar to that. Instead of sending an INSERT query, agents send a DELETE query. And unsubscribing and the actual deletion are are reversed, so that the DF can extract relevant agent information before that information is deleted from the ontology in GraphDB.

<br clear="left"/>
<br>

_Important note_: When deploying your flows in Node-RED, you always have to register the gripper agents again. While their descriptions are permanently stored in GraphDB, the dynamic MQTT subscriptions are lost. That means if you redeploy, the agents wouldn't listen on their topic.

### Sensor A - Detect and store workpiece information in GraphDB

The flow of Sensor A is basically unchanged. The detection start and end time is stored and passed to the conveyor to get the workpiece length. The only difference is that the workpiece length is now stored in GraphDB. In order to store a product with its ID and length, the following query is used. It is defined in the function `store product in GraphDB` and then sent to the GraphDB with the subsequent HTTP request.
```SPARQL
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX agents: <http://www.hsu-hh.de/aut/ontologies/agent-summer-school#>
INSERT DATA {
    agents:Product_${msg.payload.barcode} a agents:Product;
        agents:hasProductId "${msg.payload.barcode}";
        agents:hasLength ${msg.payload.wp_length};
        agents:timeAdded ${msg.payload.sensorA_timeStartDetection}.
}
```
This query is a simple INSERT query with a little twist: Its not static but instead, data of the current flow execution is used. The syntax `${msg.payload.barcode}` basically inserts a JavaScript variable - in this case the barcode of the workpiece - into the query. And just like that, workpiece length and a timestamp are also added. With this query, all incoming products are added into our ontology in GraphDB.

### Conveyor - Calculate workpiece lengths
The conveyor is completely unchanged. It awaits requests (from Sensor A) and calculates the workpiece length using its conveyor speed. The workpiece length is then returned to Sensor A.

### Sensor B - Wait for products, get relevant MQTT topic and create proposals

Sensor B is where most of the logic for a proper agent interaction happens. As soon as a workpiece arrives at B, the ID is "detected" by getting the last ID generated by Sensor A. Sensor B the queries the length of this ID from GraphDB, calculates the stop position and tells the conveyor to stop (HTTP request). Sensor B then wants to contact all agents that provide a capability of type "Handling". Therefore, it asks the DF for all the MQTT topics that may be published to. 
Recall that in our case, the DF puts all agents in a topic according to their capability type. So there is only one topic for handling - the "Handling" topic. This topic is returned to Sensor B.
![image](https://user-images.githubusercontent.com/50097079/178676524-caae40d8-fb92-4f0f-bd9e-75a7f3fc3692.png)
As there could also be more than one topic, SensorB separates the topics and sends a call for proposal (CFP) to every topic. After sending the CFP, Sensor B waits for incoming proposals. We will get to this handling proposals later. Lets first look at how the grippers react on the CFP sent by Sensor B.


### Grippers - Reacting on CFPs
![image](https://user-images.githubusercontent.com/50097079/178677999-eef746ca-5e8a-438b-a613-66bd496dd670.png)
When registering, both grippers are told to subscribed to the handling topic by the DF (read above). When Sensor B publishes messages through this topic, they are received by all registered grippers. The grippers then check the performative of the message. In case of a CFP, every gripper checks has to check whether its own capability is suited to fulfill the task requested by Sensor B. In our use case, the task is to grab a product with a random workpiece length from a given position (with X, Y and Z coordinates) and place it on a target position.
<br>
In order to check whether or not a gripper's capability is able to fulfill this task, the gripper gets its own capability description with all the properties and their constraints. Here you can see the query that Gripper A uses (Gripper B similar):

```SPARQL
PREFIX agents: <http://www.hsu-hh.de/aut/ontologies/agent-summer-school#>
PREFIX VDI3682: <http://www.hsu-ifa.de/ontologies/VDI3682#>
PREFIX DINEN61360: <http://www.hsu-ifa.de/ontologies/DINEN61360#>
SELECT ?elem ?type ?expGoal ?exp ?url WHERE {
    agents:Gripper_A agents:hasCapability ?capability.
    ?capability VDI3682:hasInput ?elem;
    	agents:hasUrl ?url.
    ?elem DINEN61360:has_Data_Element ?de.
    ?de DINEN61360:has_Type_Description ?type;
             DINEN61360:has_Instance_Description ?inst.
    ?inst DINEN61360:Value ?val;
                   DINEN61360:Logic_Interpretation ?logInt;
                   DINEN61360:Expression_Goal ?expGoal.
    BIND(
        CONCAT(STR(?expGoal), STR(?logInt), STR(?val)) AS ?exp)
}
```
Basically, all capabilities of Gripper A are searched together with their input constraints. Every constraint consists of an expression goal (= "Requirement" for inputs), a logic interpretation (a relation operator such as "<", "<=",...) and a value. These things are concatenated together and stored into the ?exp variable in the last line of the query. 
As a result, ?exp will contain values such as `"Requirement <= 50"` or `"Requirement > 10`. The ?type variable stores the type of property that each expression is about, i.e. workpiece length or X/Y/Z position.
Inside the function "process results", the grippers take the current values for workpiece length as well as the positions and replace the word "Requirement" with the value of the respective type.
Lets say we have an expression of "Requirement <= 0.75" for the type "Length" and a current workpiece length of 0.9, then this replacing would lead to the expression "0.9 <= 0.75". Using JavaScripts `eval()` function, this expression is evaluated. In this example, it would obviously return `false` indicating that the workpiece is too large to be handled by this agent's handling capability.
With the "process results" function, all properties are checked and only if all property expressions are evaluated to `true` is a capability suitable. Only in this case, a proposal is created and sent back to the agent that sent the call for proposals (Sensor B in our case).

### Sensor B - Waiting for proposals
In the previous section, we looked at how agents (grippers in our case) receive CFPs and react on those CFPs. At the same time, Sensor B is waiting for proposals. A timeout of 5s is set and if those 5s are over, Sensor B stops waiting for proposals. During those 5s, Sensor B checks for incoming proposals every second. This is done using a trigger node. 
All received proposals are checked. If all expected proposals are received or if the timeout is hit, the trigger is reset, which stops Sensor B from waiting on further proposals. 
Sensor B takes all the received proposals, determines the best proposal by looking for the cheapest price. It accepts the cheapest proposal and rejects all others.
![image](https://user-images.githubusercontent.com/50097079/178682998-41e9003a-b835-449f-803d-4762834392ff.png)

In order to actually receive proposals, Sensor B is listening for its own MQTT topic which is passed with the CFP so that all agents can respond. For every proposal that arrives via MQTT, Sensor B checks whether it belongs to the current CFP by comparing message IDs. If it is a valid proposal for the current CFP, it is stored and processes as just described above.
![image](https://user-images.githubusercontent.com/50097079/178684050-7c8167cb-5e6b-4807-bbc1-03e2197a8175.png)


### Grippers - Reacting on accept / reject
Grippers use the "check performative" node to react on different message types. In the section [Grippers - Reacting on CFPs](https://github.com/hsu-aut/ISSIA22-node-red-agents#grippers-Reacting-on-CFPs), you can see how CFPs are handled. In case of accepts or rejects, the grippers first check whether or not the current message ID exists and if a proposal was sent. If it's an unknown ID, a error message is send to the console. If it's known, the ID is deleted. 
In case of an accept message, the actual gripper handling logic is executed. This handling operation is simulated by just waiting for a second and then sending a "finished" message to the console.

![image](https://user-images.githubusercontent.com/50097079/178684618-fcaa6007-85dc-4778-9506-00f60619dcef.png)


## 🚨 Limitations 🚨
- The start and target positions for the gripping capability are not calculated dynamically, but instead are just contained inside a global configuration. We did this to keep the message object small and understandable. Ideally, the position would be dynamically calculated based on a base configuration of the conveyor together with the workpiece length and stop position. The positions should also be sent together with the workpiece length inside the CFP.
- The grippers don't check their output constraints when evaluating a CFP. Currently, only inputs are considered to keep the query a bit shorter and easier. Ideally, every gripper would have to check whether input requirements are fulfilled AND output requirements can be met.
- The price is currently a simple constant value set by the grippers. Ideally, it would be calculated dynamically by taking into account the time it takes a gripper to move a workpiece from start to end point.
