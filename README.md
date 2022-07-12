# ISSIA22: Node-RED Software Agent Workshop
Workshop on how to develop software agents using Node-RED and semantic web technologies held during the 2022 International Summer School on Industrial Agents

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

### Approach 1: Local data storage and direct communication
The flow of Sensor A starts with an inject node that is used to simulate an arriving workpiece. The inject node creates an empty payload as well as a random msg.delay. In the subsequent function node, an ID is generated (= simulating detection) and the start time is set. The delay node takes the msg.delay that was in the inject node and pauses the execution for the given delay value. This simulates the workpiece passing through the sensor.

Then, the time difference between start and end of detection is calculated and used to request the length from the conveyor. We assume that only the conveor knows its speed and thus Sensor A cannot calculate the workpiece length by itself. Sensor A uses an HTTP request node to request the workpiece length from the conveyor. After getting a response, Sensor A stores the workpiece length locally (i.e. in the flow using `flow.set()`).

![image](https://user-images.githubusercontent.com/50097079/178309446-c1343b92-611f-494e-a6c5-cc24c856ffe6.png)

The inject node of Sensor B is used to simulate arrival of the workpiece at B. The last workpiece ID is looked up (it is stored globally to simulate workpiece order). Sensor B requests the length of the workpiece with the given ID from Sensor A. After getting the length, B calculates the time it takes to stop a workpiece at half its length and requests the conveyor to stop. As soon as the conveyor stops, Sensor B selects a gripper using a simple if-condition. The selected Gripper is then requested to handle the workpiece. 

![image](https://user-images.githubusercontent.com/50097079/178311903-6ddd3ee2-ab97-4e3a-ba80-811abe77534e.png)

Please note that this approach was just discussed, but not fully implemented during the workshop. You can try it out by importing the flows inside the `flows\local` folder. Make sure to also import the conveyor.

### Approach 2: Using an ontology and more detailed agent communication

With this second approach, we want to increase flexibility of the overall system and we want to use an explicit model of agents, their capabilities as well as constraints on how & when capabilities may be used. Furthermore, we also want to add a more "realistic" kind of agent communication. MQTT is used to dynamically assign groups of agents and communicate with them without the need to create a direct connection.
For this approach to work, you have to set up a Graph DB instance on your machine. This instance has to have a repository with name + ID = "AgentSummerSchool" into which the ontology `AgentSummerSchool-Ontology_merged.owl` has to be imported. 

<img src="https://user-images.githubusercontent.com/50097079/178440843-f6caf33f-6f1f-424e-b3fd-e9c6b4ab77da.png" align="left" width="400px"/>

We start by providing a way for agents to register themselves and their capabilities by clicking on an inject node. To achieve that, a Directory Facilitator (DF) agent is added, which acts like a registry. 

It allows agents to register themselves and manages MQTT topics. Agents can send a SPARQL INSERT query in their msg.payload to the DF via HTTP. The DF will forward the query to the GraphDB effectively registering the agent. It then takes the msg.payload.sender to retrieve information about the agent it just registered. The capabilities of the agent are taken and used as MQTT topics. The DF sets the topic by sending an HTTP request back to the agent.

![image](https://user-images.githubusercontent.com/50097079/178442301-401ceb71-cb50-4c8f-81a3-25631e841b2d.png)

<img src="https://user-images.githubusercontent.com/50097079/178442839-f663c0a4-c3bf-4bde-bf35-9ef1a51173d6.png" align="left" width="400px"/>
In order for the DF to be able to pass the MQTT topic back to an agent, agents (like Gripper A and B) provide an "http in" node. The url is contained in the agent description (i.e. the INSERT query) and can thus be dynamically found by the DF. In this way, both Gripper A and Gripper B can be registered. Deregistering works just similar to that. Instead of sending an INSERT query, agents send a DELETE query. And unsubscribing and the actual deletion are are reversed, so that the DF can extract relevant agent information before that information is deleted from the GraphDB.

<br clear="left"/>
<br>

_Important note_: When deploying your flows in Node-RED, you always have to register the gripper agents again. While their descriptions are permanently stored in the GraphDB, the dynamic MQTT subscriptions are lost.

🚧 Documentation of the following steps coming soon 🚧
- Sensor A now adds all lengths to the GraphDB
- Sensor B gets all relevant topics from the DF
- Sensor B then contacts all topics and sends out a CFP
- Grippers get this CFP and check whether all constraints are fulfilled
- If fulfilled, they create an offer
- Sensor B waits for all offers
- After timeout or all offers received: Sensor B decides on the best offer
- Sensor B sends accept to best offer
- Sensor B sends reject to all others
- Grippers delete conversation id and execute if offer got accepted
- Grippers delete conversation id if offer got rejected



