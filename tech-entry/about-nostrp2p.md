# **NostrP2P: Pure P2P Distributed Microblogging System**


Hello. This is ryo_grid.

This time, I created a pure P2P distributed microblogging system called NostrP2P, and I will write about it.

- For now, the GitHub repository of the development is here:
  - [ryogrid/nostrp2p](https://github.com/ryogrid/nostrp2p)
  - [ryogrid/flustr-for-nosp2p](https://github.com/ryogrid/flustr-for-nosp2p)


![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/12325/9fce752c-3d73-fa48-10ea-0770330a3b38.png)

## Prerequisite Knowledge
- A rough understanding of the [Nostr](https://github.com/nostr-protocol/nips) protocol
  - You can refer to [wikipedia page](https://en.wikipedia.org/wiki/Nostr) and [this page](https://nostr.how/en/the-protocol) for sufficient understanding

## Background of Development
- Originally, I wanted to create a pure P2P application that works on an overlay that traverses NAT
  - I once created a [DHT-based distributed KVS](https://qiita.com/ryo_grid/items/9a82d7230fbc4a0875c1), but it could not traverse the NAT wall
- From the above idea, I wondered if I could roughly implement a NAT traversal overlay with a gossip protocol, etc.
  - => I found [weaveworks/mesh](https://github.com/weaveworks/mesh), which is not only that but an even more intelligent implementation
- mesh is very well made, but it could not establish reliable connection-based communication between nodes. Additionally, it had some quirks in usage, and it felt like I had to go through a lot of trial and error to use it
  - => I developed [ryogrid/gossip-overlay](https://github.com/ryogrid/gossip-overlay), which is something like a wrapper library for mesh that constructs a reliable communication channel on the overlay NW created by mesh using the SCTP protocol
- Around the time I started messing with mesh, I learned about the Nostr protocol, which is used for implementations of distributed SNS (microblogs)
- Additionally, with the various things about Twitter becoming X, I also looked into other distributed SNS
- As a result, I thought the following:
  - Wouldn't the various burdens on server operators eventually make these bankrupt?
    - In Misskey.io, it was [in this state (reffered post is wrote in Japanese :) )](https://misskey.io/notes/9brqui38ib)
    - Although the above is an extreme example, overall it seemed to depend on volunteer resources, which, at least to me, did not seem very healthy
- All the above mixed together, and since the timing coincided roughly, I decided to create the NostrP2P mentioned in the title
  - (One reason was that I couldn't find an implementation of distributed microblogging with pure P2P as far as I searched. If anyone knows, I would be grateful if you could let me know)

## Concept of NostrP2P
- **A system composed of the contributions of all users**
  - Issue: The design of existing distributed SNS (Mastodon, Nostr, Bluesky, etc.) places a high financial and operational burden on server operators, and when viewed as a system as a whole, it lacks soundness (as it seems to me)

## Features of NostrP2P
- A system centered around broadcast using the gossip protocol
  - Uses the [weaveworks/mesh](https://github.com/weaveworks/mesh) library, which enables the construction of an overlay network and various types of messaging on it, as the communication infrastructure
- Focuses more on ease of implementation and simplicity rather than performance and data consistency
  - The rationale may ultimately be to reduce workload, but adding complex mechanisms in such systems makes it challenging to operate stably
  - For the above reasons, structured mechanisms such as DHT are not adopted (for now)
- Operates overall in a fuzzy manner (e.g., allows for minor message loss)
  - Unlike other distributed SNSs, it operates on a pure P2P architecture that is challenging to achieve reasonable performance, so it does not offer many rich features
- Each server operates on the overlay network
  - Server on private network separated with Internet by NAT can be join 
  - NAT traversal is achieved through relays by servers with global IPs
- Uses the concepts and data structures of the [Nostr](https://github.com/nostr-protocol/nips) protocol as its foundation
  - Reason for adopting Nostr protocol as based design
    - Nostr protocol is relatively easy to implement
    - Ecosystem of Nostr protcol can be used
    - It might be interesting to work with other systems or toools based on Nostr protocol
  - Utilizes public key-based data signatures for authentication, designs various messages for microblogging applications, and structures message data
  - However, due to the different underlying architecture, it is not compatible as a result of optimization
- Each user sets up their own server. Clients primarily communicate only with their own server
- To keep the communication volume low, it serializes data into a binary format where necessary, especially for parts with high traffic, and optimizes for the application and architecture
- Emphasizes push over pull (details later)
- Assumes clients are mainly used on mobile devices like smartphones, optimizing for communication volume and power consumption, which are particularly important on these devices
  
  System configuration concept diagram  
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/12325/83fc6146-0e15-a363-ddac-d3d373c21c48.png)

## Details (Design and Implementation)
- Each user sets up their own server, and client access is limited to their own server
  - If the server is set up on a home machine (i.e., a machine without a global IP), direct access from mobile devices over mobile networks is not possible, but it can be handled by setting up a VPN with tools like Tailscale
  - **However, without servers operating with public IPs, the overlay system cannot function**
    - => For servers operating with public IPs, posts from clients must be signed, and the server verifies the signature
- Emphasizes push over pull
  - Broadcast and have it received only by followers, discarding for others
    - The assumption is that recipients are online during broadcasts
- Designed based on the premise of using the mesh library in Go language
  - (Unfortunately, there are no library implementations for connecting to the overlay network formed by mesh in languages other than Go. There are no other options either)
  - In mesh, each node has a 64-bit ID
  - Each node knows the IDs of all nodes participating in the overlay network
    - There is a delay in updating, but it should be within 2-3 seconds for up to 100 nodes I think
    - By using the lower 64 bits of the public key as the node (server) ID, there is no need to look up the node ID corresponding to the user
- Private keys
  - Only the client needs to have it
  - The server only needs the public key. With this information, it can reject posts from anyone other than the user who has the corresponding private key, and users can be identified by the node ID associated with the public key, allowing other servers to unicast messages
  - The private and public keys of Nostr can be used as they are
- Do not accumulate long-term data
  - (Except for profile information, etc.) There are few opportunities to refer to old data unless it is a special client, so we cut it off boldly
  - This keeps the server's memory usage and storage usage low. Also, having less data means that the server's processing load when searching for requested data is kept low
- Communication between servers is done in binary, but the structure of the data exchanged is the same as the Nostr protocol
  - MessagePack is used for the binary format. ProtoBuf was also tried, but it was not adopted because the change in data size was small compared to the effort required
  - We do not use over TLS for communication as it increases the hassle of setting up servers
    - It seems that mesh can encrypt using a common key generated from a commonly set password in the application, but due to concerns about performance degradation and incompatibility with OSS, this feature is not used
    - Even if we do, it would be done separately with E2EE, but there is a challenge that it is not compatible with broadcasting...
- Client
  - Communicates with its own server via REST I/F
    - The basic data format is JSON text
    - Responses to data requests are returned from the server in serialized binary with MessagePack
    - Lower communication frequency with the server allows mobile devices to rest the antenna for mobile networks, reducing power consumption, and for the server, sending a certain number of data together increases the effect of gzip compression at the HTTP layer, so the current implementation sends a request to the server every 10 seconds, and the server sends the data received from other servers after the previous request together
  - As mentioned above, when adding or updating information related to the user, such as posting from the client, a signature is attached
  - Conversely, in the design of NostrP2P, the user's own server can be trusted, so the client does not verify the signatures of received data. Therefore, the server can send the signature information empty to reduce communication volume

## Supported features
- Posting
  - Currently, to keep the implementation simple, broadcast to all users on the overlay NW
    - Honestly, if the number of users reaches the order of hundreds, this design may have its limits
    - In this case, it would be necessary to manage follower information and modify it to switch to multicast instead of broadcast. Also, by forwarding in a tree structure, it may be necessary to prevent the number of servers each server directly sends to from increasing
- Profile
  - Broadcast when updated
  - If there is no profile information (including user icon and username) to display along with the post when the client tries to display it, it will send a request to its own server. If its own server cannot respond, it will send a request to the corresponding user's server. If the server is offline, it will resend the request later
- Follow
  - Currently, the only way to follow a user is finding and following the user in the global timeline
    - (If the client is elaborated, it is possible to provide a UI to follow a user with a specified public key)
- Reply (mention)
  - Can be done to anyone if the post is visible, but the interaction is only visible to the parties involved
- Like (Favorite)
  - Unicast to the user's server who posted it
    - If the recipient's server is offline when sending, it will resend later
  - The liker can only see that they liked the post and not the total number. The liked user can see the total number
  - Other users cannot know any information about the like situation
- Repost & Quote Repost
  - Unlike replies, broadcast to all users
  - If the client has not received the post that was quoted repost, it will request it from the server. If the user's own server does not have the requested post, it will send a request to the server of the user who posted it. If the server is offline, it will resend the request later
- Hashtags
  - Not supported as it would increase the load of post search processing on the server and generate queries to the entire NW
- Deleting Posts
  - Not implemented at this time
  - As long as the current design, which broadcasts posts to all servers, does not face scalability issues, it is possible to implement a design where a deletion request is sent in the same manner, and the receiving server deletes the data
    - However, if a server that has customized the author implemented one or an implementation developed by someone other than the author appears and ignores the deletion request, the post will not be deleted
    - Data on servers that were not online when the request was sent will also not be deleted
      - Even if the request is sent multiple times, if the timing is not right, it will not work, so in the extreme case, there is no guarantee that the data will be deleted
  - Given that the servers are distributed and no one has the authority to enforce privileged operations, it is safest for users to assume that once a post is made, it cannot be deleted

## Demo
Try connecting to the demo server with the web client.
If it works, it should display as shown below.

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/12325/9fce752c-3d73-fa48-10ea-0770330a3b38.png)

### Web Client
- [https://nostrp2p.vercel.app/](https://nostrp2p.vercel.app/)
  - Made with Flutter
  - Recommended to access with Chrome on PC, smartphone, or tablet

### Client settings
Click/touch the gear mark at the top right of the screen and perform the following on the displayed screen.

- 1) Set the following for the Trial account as the private key
  - nsec1uvktv4u3csltg98caqzux3u0kawxz3mppxjqw40lcytqt52kdslshwr2xp
- 2) Set the server address
  - Enter the following address of the demo server
  - https://ryogrid.net:8889

**Note: It seems that the save button may not save input content well. If the input content is not displayed above the input area with the display of current server address, the setting has not been reflected, so if it is not displayed, try pressing it several times.**

Since it would be boring if you cannot write, it is possible to post and change profiles only with the Trial account.

**However,** NostrP2P is originally intended to post by communicating with your own server, and using the demo server is an exceptional operation.
If you want to have your own account, you need to set up your own server, connect it to the demo server (which also serves as an overlay NW bootstrap server), and perform operations such as posting through your own server.

## Access NostrP2P with your own account

### Step 1: Set up your own server
- Please refer to the Examples-Server Launch section in the [NostrP2P GitHub repository](https://github.com/ryogrid/nostrp2p) for how to set up the server
  - There is an option **-b** among the command line options, specify **ryogrid.net:8888** for the demo server
- private key and public key for using NostrP2P can be generated as follows
  - $ nostrp2p-server.exe genkey
    - When your OS is Windows
- Built binaries of the server are placed at the following.
  - [https://github.com/ryogrid/nostrp2p/releases/tag/latest](https://github.com/ryogrid/nostrp2p/releases/tag/latest)
  - If there is no binary for the platform you want to run, please build it yourself
- Note that the private key and public key will be different from those for the trial account
  - The key starting with nsec is the private key, and you should manage it so that others do not know it. If someone else knows it, they can post on your behalf on NostrP2P
    - The private key for the Trial account in the demo is public for demonstration purposes, which is a special case

### Step 2: Accessing Your Server from a Client
- The method slightly varies depending on the client type
  - The port number to access will be the one specified with the **- l** option when starting the server plus 1
    - For example, if the **-l** option is omitted, it defaults to 127.0.0.1:20000, so the port number for the client will be 20001
- The following section assumes the server is set up within a private network, not on a machine with a global IP address
  - Those who set it up with a global IP address accessible from the internet may not need explanations on TLS
    - However, NostrP2P servers can handle TLS communication by placing fullchain.pem (certificate public key) and privkey.pem (private key) with the same names in the current directory at server startup and adding the option '-s true'
    - Use this if you do not use reverse proxies that enable TLS communication

#### Using the Web Client ( https://nostrp2p.vercel.app )
- Due to security restrictions of web browsers, communication will be blocked if the server's REST I/F is not encrypted (not over TLS)
- Therefore, you need to enable over TLS, meaning you need to turn the REST I/F open over HTTP into HTTPS
- There are other methods, but one way to do this easily in a private network is by using the reverse proxy feature of Tailscale (a free VPN setup tool/service)
  - [tailscale serve command - Tailscale Docs](https://tailscale.com/kb/1242/tailscale-serve)
  - By using this, you can access your server from devices participating in the VPN using the demo client
  - To simplify, introduce Tailscale and refer to the above article while mapping access to "/" to "http://127.0.0.0: **mentioned port number** /" , then set the server address in the client to the URL assigned by Tailscale, "https://hoge.tailedXYZ.ts.net/"

#### Using Native Clients
- When using the various native clients available, the REST I/F of the personal server can remain as HTTP
  - [https://github.com/ryogrid/flustr-for-nosp2p/releases/tag/latest](https://github.com/ryogrid/flustr-for-nosp2p/releases/tag/latest)

## Challenges in Design and Development
- Struggled with design decisions
  - Making replies visible only to participants, and favorites known only to recipients, to reduce inter-server communication, took time to decide
- Client implementation was challenging
  - Took more time than server implementation
  - Referenced the [uchijo/flustr](https://github.com/uchijo/flustr), a Nostr protocol-based microblog client, modifying and enhancing it for NostrP2P
    - Grateful for uchijo's great work
    - Familiar with the Flutter framework used in Flustr, it was a good fit as I am not fond of web frontend frameworks
    - Struggled with understanding Riverpod library, used for state management
  - Although there are many third-party libraries for Dart/Flutter, many do not support web builds
    - For example, the HTTP2 library does not support web builds, so HTTP2 is not used
    - Flutter Web's stability is still lacking, I did some workaround solutions

## Remaining Issues
- The GitHub repository of the mesh library say NW could scaling up to about 100 nodes, but it may mean that mesh library is not usable for larger scales
  - Although it performs relatively intelligent routing, it might struggle with routing decisions and updating node information with more nodes
  - If 100 nodes is the limit, developing more scalable communication foundation might be necessary
- Broadcasting posts to all users may be impractical with more servers
- While signature verification prevents impersonation, multiple servers starting with the same public key can cause message delivery issues
- The network could become dysfunctional if someone sends large amount of messages, and currently, there is no countermeasure against such attacks
- Running a server 24/7 might be difficult for users who don't already operate a server, limiting the potential user base
  - Using cheap Android smartphone as server machine may be a solution!
    - [NostrP2P server on Android Smartphone](https://gist.github.com/ryogrid/a8c585a7d229a666ff82369df1939058#file-nostrp2p_on_termux-md)
- Can you set up a server that is either outside the NAT or inside the NAT but accessible from outside the NAT?
  - To allow communication between servers inside the NAT, a relay server like the one mentioned above is necessary
  - Additionaly there's a concern that if there are no other bootstrap servers (servers like the one mentioned above that have their address published) besides the ones I have published, it could become a single point of failure.
    - In case of a downtime or maintenance, it won't affect servers that are already connected, but new servers trying to join or servers trying to reconnect will face issues
  
**Enjoy!**
