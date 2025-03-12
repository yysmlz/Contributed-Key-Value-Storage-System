# raft-java
Raft implementation library for Java.<br>
Based on the [RaftPaper](https://github.com/maemual/raft-zh_cn)and the Raft open-source implementation[LogCabin](https://github.com/logcabin/logcabin)。

# Supported Features
* Leader election
* Log replication
* snapshot
* Dynamic cluster membership changes


## Quick Start
To deploy a 3-instance Raft cluster locally, run the following script: <br>
cd raft-java-example && sh deploy.sh <br>
This script will deploy three instances (example1, example2, example3) in the raft-java-example/env directory.<br>
 It will also create a client directory for testing Raft cluster read/write functionality.<br>
After deployment, test the write operation by running the following script:
cd env/client <br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world <br>
 Test the read operation with the command：<br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

# Usage
Below is the instruction to use the raft-java dependency library to implement a distributed storage system in your code.
## Configuring Dependencies (Not yet published to Maven Central Repository, if demployed, need to install manually)
```
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Define Data Write and Read Interfaces
```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server Side Usage
1. Implement the StateMachine interface class
```java
// This interface includes three methods mainly for internal use by Raft
public interface StateMachine {
    /**
     * Write a snapshot of the data in the state machine, called periodically by each node locally
     * @param snapshotDir Directory to output snapshot data
     */
    void writeSnapshot(String snapshotDir);
    /**
     * Read the snapshot into the state machine, called when the node starts
     * @param snapshotDir Directory containing snapshot data
     */
    void readSnapshot(String snapshotDir);
    /**
     * Apply data to the state machine
     * @param dataBytes Data in binary format
     */
    void apply(byte[] dataBytes);
}
```

2. Implement the data write and read interfaces
```
// The ExampleService implementation class should contain the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```
// Data writing logic
byte[] data = request.toByteArray();
// Replicate data to the Raft cluster synchronously
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```
// Data reading logic, implemented by the application’s state machine
Example.GetResponse response = stateMachine.get(request);
```

3. Server startup logic
```
// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Apply state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, such as:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register services for inter-node communication in the Raft cluster
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register the Raft service for client calls
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register application-specific services
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start RPCServer, initialize Raft node
server.start();
raftNode.init();
```
