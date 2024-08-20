
## **Assumptions**

### **Storage Server**

- IP address and port of naming server is known to all storage servers.
- Only text files are stored.
- Python script `setup_ss.py` and `start_ss.py` are just for helping in testing, it just creates an arbitrary number of copies of storage servers. Run `make n=5` to make 5 copies of storage server.
- Python script also assigns each server a unique port to communicate with NFS and to communicate with Clients, and also a unique SS_ID; and list them down in ss_config.txt (Assigning each SSi the port number for NFS = 1000 x (i + 1) + 500 and for client 1000 x (i + 1) + 501) assuming all the port numbers fall under the permissible range otherwise just change the port number assigning logic to accomodate more storage servers or just assign them manually in the header file. (Using 500 offset because ports till 1024 are reserved)
- Remember to change the operating system name in start_ss.py before running it
- The outer make file it to run both the python scripts together run as `make n=3` where n specifies the number of storage server you want to create and to delete all the files again just run `make clean n=3` it would clear all the SS folders and related files.
    
    ***Make sure that the value of n is same in both the cases.***
    
- Using TCP sockets for all the communication.
- Assuming that there are no 2 paths with even some common subpath. (All the file as well as folder names would be unique).
- Assuming if a file is being copied, there won't be already a file with exactly the same relative path, and if there is it's data will be overwritten.
- Assuming all the files ends with .txt (all are text files) and no folder name ends with .txt
- Got the basic tries code from chatGPT prompt : "Write me a tries code in C to store directory structure such that each node of trie contains name of a file/directory and points to next files in the directory. Write code for effective search, insert and delete", and then changed it according to our requirement.
- Manually can't change backup folder.
- If we have a very large text file then while backing it up only data which can fit into one request.
- If a storage server is down the all the paths of that storage server is still accessible but only in read mode.
- Files and folders are assumed to have all permissions (0777).
- Not all the SS are down (original + backups) at the same time.
- Maximum number of paths in a storage server is fixed and can be changed by changing the value of macro in the header file.
- ***No file/folder outside the base folder can be accessible.***

### **Naming Server**

- The NS listens to new servers , clients on a single port (2000) using TCP socket communication.
- The NS is exposed to only directories and text files.
- The NS can support only a predefined number of clients at a particular instance(100 in our case) to ensure stability and accuracy in performance.
- The NS stores backups of storage servers (once the number of servers exceed two) in two other storage servers picked according to their order of arrival.
- The NS can maintain redundancy of files but the rate of data change should not exceed a certain limit which results in loss of redundancy.
- The NS acts as a client to the storage servers on the dedicated port on which the storage servers bind on.
- The NS monitors connection status of storage servers by constantly monitoring their response to a certain "PING" request sent at regular time intervals(5 seconds).

**Client**

- The data that would be provided to be written or appended onto a file will always be less than the size of the data packet that is being sent.

## **Introduction**

### **Naming Server**

The Naming Server(NS) servers as the central hub between the clients and the Storage servers. This is the central hub of file sharing and management for the clients to communicate with. The NS has the following key features:

- It acts as an address resolver for clients to perform operations like Read,Write,Retrieve Info on the files in the storage server. It resolves the IP address and port number of the storage server for the client to communicate.
- It allows the operation of privileged operations like Creating , Deleting and Copying Folders among storage servers by directly sending requests to the Storage Servers.
- It ensures data redundancy and reliability of information by creating backups of all the information present in the storage servers and updating the backups continously.
- It supports multi-client operation ie can cater to multiple client requests concurrently.

### **Storage Server**

The Storage Servers (SS) acts as the data stores In our distributed file system implementation in C, Storage Servers play a pivotal role as the backbone of the Network File System (NFS). These servers shoulder the critical responsibility of handling the physical storage and retrieval of files and folders within the network. Tasked with the management of data persistence, Storage Servers ensure that files are stored securely and efficiently, forming the bedrock of reliable file storage and access. By distributing data across multiple servers, our system aims to enhance performance, scalability, and fault tolerance, contributing to a robust and seamless file management experience for clients connected to the network. The Storage Servers, in essence, act as the guardians of data integrity, facilitating a distributed and resilient file storage infrastructure.

### **Client**

The client serves as a user interface to communicate with the Network File System, offering several essential functionalities:

- Users can initiate a variety of requests such as Read, Write, Append, and more.
- Robust error handling mechanisms are implemented on the client side.
- The Network File System supports concurrent usage by multiple clients.
- A manual (MAN page) is provided on the client side, offering a comprehensive guide to the input formats for all commands.

## **Implementation**

## **Flow of Control**

### **Naming Server**

- The Naming server initialises all necessary parameters and starts by binding to port 2000.
- Storage servers send initialisation requests to the Naming server which is handled by registering the storage server in the NS database and storing it's details(IP Address , NS Port , Client Port , Accessible paths).
- After registering storage servers , clients can connect to NS to communicate in the file sharing system.Each client sends a request packet. For each client/storage server request a new thread is created which handles the process and ends the thread.
- Each storage server is assigned a thread in which the NS constantly tries to connect and "Ping" the storage server to see whether the storage server is responding.
- Each client is assigned a unique client_id which refers to it's specific socket used for communication.
- A separate thread handles backups and synchronization of backups.
- After performing each request , the client is informed about it's status by sending a response message with Status code which helps the client to know the status of the request.
- The NS supports the following type of requests , Read File , Write File , Create Folder , Create File ,Delete Folder.Delete File,List paths ,Fetch file information.
- To communicate with the SS , the NS connects on the dedicated NS port of the SS and performs TCP communication with the SS till the socket is closed.
- There are a predefined set of error codes which are sent throughout the system based on the type of error recorded.
- LRU caching is used to achieve higher efficiency while dealing with higher times occuring requests to increase response time(latency).
- Trie data structure is used for more efficient search for accessible paths in the storage servers.
- A logging mechanism records all TCP communication happening from the NS for transparency and debugging.

### **Storage Server**

- On initializing it first takes the list of accessible paths as command line input from the user.
- It then finds the list of not accessible paths (basically all the paths in the tree from the root folder except the accessible paths provided), and all the subsequent paths that would be added either through the client or by the SS admin manually will be counted as accessible.
- Then the SS sends the registration request to the NS with it's ip and port details.
- After registration it starts two threads to keep listening on two ports of client and nfs registration respectively.
- In the threads we are accepting connections in a while loop and on each new connection a separate thread is created to serve that request. Only a certain number of threads can function concurrently and if all the thread slots are busy any new request will directly be declined.
- In the request handler thread the SS first checks the type of request received and then serves the request accordingly and sends ACK if the request is processed successfully and sends error code and message if some error occurs while serving the request. All the backup related does not send an acknowledgement as backup has to be done asynchronously, so in case the backup request fails due to some reason the NS would not know about that and the data may become inconsistent.

### **Client**

- The client initially gathers user input, determines the type of request, processes it, and provides either the specific output or an error code.
- For Read, Write, Append, and Info requests, the client obtains the IP address and port number of the designated storage server. Subsequently, it interacts with the specified storage server to execute the requested operation.
- In the case of Copy, Create, Delete, and List requests, the client communicates directly with the naming server to retrieve the desired output.
- The "MAN" command, exclusive to the client, serves as a comprehensive command providing users with a detailed guide on the input formats for all commands.

### **Backup and Redundancy**

- To ensure Backup and Redundancy , the Naming Server has a dedicated thread which checks whether each storage server is backed up and if it is not , it finds two other online servers to back up data. Once two servers are selected,it sends a "Copy request" for all the accessible files present in the Storage server and sends a "Paste request" to the two servers selected for backing up data.This ensures each file is backed up properly in the storage servers. It also maintains where each storage server is backed up in.
- To ensure redundancy , after a storage server disconnects , it rematches all backup files present in it with files present in the storage servers. Any new files , modifications , deletions are appropriately passed as separate requests to the Storage servers.
- Whenever a new path is added/deleted from a storage server , it's backups are informed about the same performing appropriate create/delete modifications in the backup.
- Any request which modifies a file (Write , Create , Delete) is redundantly replicated in the corresponding backup files as well to ensure redundancy and fault tolerance in the file sharing ecosystem.

### **Dynamic Paths Updation**

- Initially when the SS starts a list of accessible paths is provided as command line arguments. All the paths except those provided which are present in the directory tree rooted at base folder are then considered to be unaccessible so whenever a new file/folder is created manually by SS admin or by client then it will belong to the list of accessible paths.
- In SS there is a thread that is constantly running and checking for all the file/folder paths that are in the directory tree of the root directory. It then removes the paths from the list that are not accessible and matches the rest of the list with the old accessible paths list to find out if some new path is added or some old path is deleted. And sends the appropriate request to add/delete paths to the NS. This ensures that SS remains a dynamic storage.
- The thread keep checking this every 5 seconds and sends add/delete path request to the NS so NS is also a dynamic repository.

### **Book Keeping**

- Whenever there is some communication happening between either client and NS or SS and NS, each request is being logged by NS in logs.txt file.
- Logs can be printed onto the screen by pressing ctrl+z.
- Logging (Book keeping) helps us to debug and catch any anomaly that may be present in the code or logic of our implementation.

### **LRU Caching**

- NS maintains a cache which is just a normal array of structs in which it stores information about the requests that it served to client.
- It only stores the request that were successful.
- When NS receives a request from client it first checks in the cache if the reply to that path is already present there or not if it is present there then cache is updated and that found request is inserted at the end (End of the cache has newer requests and start of the cache has older requests, cache acts like a queue). And if the request is not found in the cache then the first entry in the cache is deleted and the new entry is inserted from the back.

### **Efficient Search**

- To implement efficient searching of paths and respective SS having the path we have implemented tries data structure to store all the file paths of the respective SS.
- If the search function of tries returns the ss_id of the storage server in which the path is present (backup storage id if it is the backup path and the original ss_id if it's own accessible path) and returns 0 if path is not found in the trie.
- The trie implementation follows the lazy deletion method by just removing the last end node and resetting it to zero.
- Each node in the trie has 256 children each for 256 1 byte ASCII characters possible.

### **Multiple Clients handling**

- To manage multiple clients, a global linked list is employed to store all the paths currently undergoing write or append operations. When a Write/Append request is received, the system checks if the specified file path is present in the linked list. If it is, the response indicates that the file is not accessible. Conversely, if the path is not in the linked list, it is added, and the corresponding function is called. After serving the Write/Append request, the path is removed from the global linked list.
- For Read/Info requests, the system first verifies whether the provided path is currently undergoing a write operation. If the path is found in the linked list, indicating an ongoing write process, the response indicates that the file is not accessible. However, if the path is not in the linked list, the corresponding function is executed, allowing the code to proceed

### **READ Request**

- FORMAT :
    - READ <file path>
- DESCRIPTION :
    - It is used to read data written in a text file.
    - Client first sends read request with the path of the file (relative path) to the NS which then searches for the file path. If the path is found with a particular SS then it returns the ip address and port number of the SS to the client, if the path is not found anywhere then NS just returns an error message saying that the requested file does not exist to the client.
    - If the SS to which the path belongs to is down then NS returns the ip and port of one of the backup servers.
    - Client then uses this ip and port to send read request to the storage server. (Format : <path>)
    - SS then reads the file whose path is sent and breaks the data into chunks and send the chunks one by one to the client followed by an ACK when all the data is sent.
    - Client keeps receiving data and printing it on the screen till the ACK (Stop packet) is received.

### **WRITE Request**

- FORMAT :
    - WRITE <file path>
- DESCRIPTION :
    - After entering the prompt the client sends a request to the NS demanding the port and ip of the SS having the path that we want to write.
    - NS replies to the client with respective ip and port if it is found in some SS otherwise returns an error message.
    - If the ip and port are received, then the client asks for data that is to be written onto the file and sends it to the SS whose port and ip are received. (Format : <path>|<content>)
    - SS opens the file in write mode and writes the data in the file and sends an ACK if write happens without any error, in case some error occurs while writing to the file SS just sends FAILED error code along with the message why the fail happened.
    - If ACK is received from the SS the client just exits normally else prints the error message received.

### **APPEND Request**

- FORMAT :
    - APPEND <file path>
- DESCRIPTION :
    - Similar to write request just that the file would be opened in append mode at SS.

### **CREATE FILE Request**

- FORMAT :
    - CREATE FILE <path> <file name>
- DESCRIPTION :
    - It first sends the create file request to the NS (FORMAT : <path>|<file name>). NS first searches for the SS in which the given path resides. If the path is not found with any SS it returns appropriate error message to the client. On the other hand if the path is found NS sends a request to the respective SS to create the file.
    - If the path to the file does not exist (intermediate files are not there) then the file won't be created and appropriate error message would be sent to NS by SS.
    - NS would just forward the packet received from SS (either ACK or Error message) and client would just print the error message if the file creation is not successfull.

### **CREATE FOLDER Request**

- FORMAT :
    - CREATE FOLDER <path> <folder name>
- DESCRIPTION :
    - Similar to create file.

### **DELETE FILE Request**

- FORMAT :
    - DELETE FILE <path>
- DESCRIPTION :
    - The client sends the request to NS in the given format. The NS detects this data and checks whether the given path is an accessible path. If it is not it returns 404 error otherwise it sends a delete file request to the storage server having the file.
    - The storage server uses a system call to remove the file from the local storage and sends an acknowledgement to the NS. The NS also sends a request to delete the corresponding file from the servers which contain the backup of the desired server.
    - The client receives acknowledgement from NS on completion of request or suitable error code that the NS forwards from the SS.

### **DELETE FOLDER Request**

- FORMAT :
    - DELETE FOLDER <path>
- DESCRIPTION :
    - Similar to delete file but if a folder has some contents , all the contents are recursively deleted from the folder.

**COPY FILE Request**

- FORMAT :
    - COPY FILE <path> <path>
- DESCRIPTION :
    - The client sends the request to NS with the source and destination. The NS sends a COPY request to the storage server containing the source file and the server responds with the content of the data.
    - The NS sends a request to destination folder with the destination path modified as destination folder followed by file name.
    - The SS creates the relevant file in the destination folder and writes the content into it.
    - The SS sends an acknowledgement on receiving data.
    - In case where the second argument is a file , the client program does not send a packet but informs the client about invalid input type *Redundancy is maintained by sending a COPY to backup storage serves as well.

### **COPY FOLDER Request**

- FORMAT :
    - COPY FOLDER <path> <path>
- DESCRIPTION :
    - Just like copy file , the client sends a request to NS with source and destination. The NS sends a COPY FOLDER request to source folder if it exits and it responds with a folder object containing all accessible paths in the folder. The NS copies files and creates directories into the destination folder from this object.

### **INFO Request**

- FORMAT :
    - INFO <path>
- DESCRIPTION :
    - Provides file size , date modified , permissions of a file to the user if it is a text file.

### **LIST Request**

- FORMAT :
    - LIST
- DESCRIPTION :
    - Lists all accessible paths which can be accessed at the moment (if a storage server is down as well , they are accessible just under read operation only).

### **MAN Request**

- FORMAT :
    - MAN
- DESCRIPTION :
    - Dont't know how to tell the client what to do ? Check this out! This tells you syntax of all available client commands!

### **EXIT Request**

- FORMAT :
    - EXIT
- DESCRIPTION :
    - Goodbye!😊