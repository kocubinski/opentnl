###### How does TNL handle this (especially the creation of the initial player object)?
###### How does TNL handle this (especially the creation of the initial player object)?
###### How does TNL handle this (especially the creation of the initial player object)?

In TNL the server creates the player on connection (e.g. TestConnection::onConnectionEstablished) 
(could also be an RPC), and makes it ghostable.  The GameObject is added to the list of ghosts 
for that particular connection, and the client will receive the update (and begin trackin the ghost
itself) on the next packet.

onConnectionEstablished looks like an event, so where is it fired off from?

for a local, short-circuit connection.  may need to check this out later for listen-server 
implementation.
    -> NetConnection::connectLocal(NetInterface *connectionInterface, NetInterface *serverInterface)
    -> void TestGame::createLocalConnection(TestGame *serverGame)
    -> createGameClientServer() [testWindow.cpp]

----

this is called into on the server
    -> void NetInterface::handleConnectRequest(const Address &address, BitStream *stream)

----

this is called into on the client in response to above
    -> void NetInterface::handleConnectAccept(const Address &address, BitStream *stream)
    
----

NetInterface keeps a list of all NetConnections
There is one NetConnection for each client.  Thus, a server has one NetInterface and many 
NetConnections, while a client has one NetInterface and one NetConnection (the connection 
to the server).  

NetInterface only handles packets relating to establishing connections or pinging.
NetInterface::processPacket dispatches the a non game info or connection packet to the 
appropriate NetConnection by looking up the source address and calling readRawPacket


The state updates are handled in GhostConnection [: EventConnection : NetConnection]
e.g. Player::unpackUpdate which implements NetObject::unpackUpdate
which is called from GhostConnection::readPacket.  Here you can see the structure of a typical
TNL state update packet (based on how its read from the BitStream)


-> NetObject::unpackUpdate

 -> GhostConnection::readPacket

  -> NetConnection::readRawPacket

   -> NetInterface::processPacket

    -> NetInterface::checkingIncomingPackets

     -> TestNetInterface::tick

    

-> NetObject::packUpdate

 -> GhostConnection::writePacket

  -> NetConnection::writeRawPacket

   -> NetConnection::checkPacketSend

    -> NetInterface::processConnections

     -> TestNetInterface::tick



Since all of the ghosts are stored in GhostConnection 

protected:

    GhostInfo **mGhostArray

    

this means that the server maintains state seperately for each connection client

so how do we ensure that state updates from one GhostConnection get updated in every other

GhostConnection ?
