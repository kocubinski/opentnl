opentnl
=======

Torque Network Library

# TNLTest

#### Creation of the initial player object?

The server creates the player on connection `TestConnection::onConnectionEstablished` (could
also be an RPC), and makes it ghostable. The `GameObject` is added to the list of ghosts for
that particular connection, and the client will receive the update (and begin tracking the
ghost) on the next packet.  

for a local, short-circuit connection.  may need to check this out later for listen-server 
implementation.
    -> NetConnection::connectLocal(NetInterface *connectionInterface, NetInterface *serverInterface)
    -> void TestGame::createLocalConnection(TestGame *serverGame)
    -> createGameClientServer() [testWindow.cpp]

`onConnectionEstablished` looks like an event, so where is it fired off from?

On the **server**:  

    void NetConnection::onConnectionEstablished()
    void NetInterface::handleConnectRequest(const Address &address, BitStream *stream)

On the **client** (in response to above):  

    void NetConnection::onConnectionEstablished()
    void NetInterface::handleConnectAccept(const Address &address, BitStream *stream)

`NetInterface` keeps a list of all `NetConnection`s. There is one `NetConnection` for each client.
Thus, a server has one `NetInterface` and many `NetConnection`s, while a client has one
`NetInterface` and one `NetConnection` (the connection to the server).  Each `GhostConnection`
is bidirectional, and each side tracks a list of all ghosts:

    /// Array of GhostInfo structures used to track all the objects ghosted 
    /// by this side of the connection.
    /// For efficiency, ghosts are stored in three segments - the first segment contains GhostInfos
    /// that have pending updates, the second ghostrefs that need no updating, and last, free
    /// GhostInfos that may be reused.   
    GhostInfo **mGhostArray;   

The list is updated continuously depending on what objects are in scope.  So a player moves
into scope, gets a particular ID, moves out of scope, and moves back in - that player will
mostly likely not have the same id.

NetInterface only handles packets relating to establishing connections or pinging.
NetInterface::processPacket dispatches the a non game info or connection packet to the 
appropriate NetConnection by looking up the source address and calling readRawPacket

The state updates are handled in `GhostConnection [: EventConnection : NetConnection]`
e.g. `Player::unpackUpdate` which implements `NetObject::unpackUpdate`
which is called from `GhostConnection::readPacket`. In those methods see be seen the structure
of a typical TNL state update packet (based on how its read from the `BitStream`)  

Receiving/unpacking an update:

    NetObject::unpackUpdate
    GhostConnection::readPacket
    NetConnection::readRawPacket
    NetInterface::processPacket
    NetInterface::checkingIncomingPackets
    TestNetInterface::tick

Sending/packing an update:

    NetObject::packUpdate
    GhostConnection::writePacket
    NetConnection::writeRawPacket
    NetConnection::checkPacketSend
    NetInterface::processConnections
    TestNetInterface::tick

Since all of the ghosts are stored in GhostConnection 

    protected:
        GhostInfo **mGhostArray

This means that the server maintains state seperately for each connection client so how do
we ensure that state updates from one `GhostConnection` get updated in every other
`GhostConnection`?  

    // notify the network system that the position state of this object has changed:
    NetObject::setMaskBits(PositionMask);
    Player::serverSetPosition
    TNL_IMPLEMENT_RPC(TestConnection, rpcSetPlayerPos, 
      (TNL::F32 x, TNL::F32 y), (x, y),
      TNL::NetClassGroupGameMask, TNL::RPCGuaranteedOrdered, TNL::RPCDirClientToServer, 0)
    ~~~~ RPC Call ~~~~
    TestGame::moveMyPlayerTo(Position newPosition)

So the bit mask gets set, but what consumes it? In `NetObject`:

    static NetObject *mDirtyList;
    NetObject *mPrevDirtyList;
    NetObject *mNextDirtyList;

This is a shock to me, does `NetObject` really track **all** dirty objects in scope for
**anyone** in a static linked-list?  Well, perhaps its not so bad.  Objects only get pushed
to the list if the client has some interaction with them.  Additional processing is done
with:

    /// collapseDirtyList pushes all the mDirtyMaskBits down into
    /// the GhostInfo's for each object, and clears out the dirty list.
    static void collapseDirtyList();

But there is also a note in the:

    /// Notify the network system that one or more of this object's states have
    /// been changed.
    ///
    /// @note This is a server side call. It has no meaning for ghosts.
    void setMaskBits(U32 orMask);
  
So if clients cannot set dirty masks, how does client-side prediction work?  TNLTest doesn't
use any.  A client clicks, an RPC is sent, and the server responds over time with state
updates.  So onto ZAP...

# ZAP

There are two main loops which drive the game,
[`ClientGame::idle`](https://github.com/kocubinski/opentnl/blob/master/zap/game.cpp#L406)
 and 
[`ServerGame::idle`](https://github.com/kocubinski/opentnl/blob/master/zap/game.cpp#L335R),
which kick off the packet read/writes just as in TNLTest.  Movement processing starts in
[`ControlObjectConnection::readPacket`](https://github.com/kocubinski/opentnl/blob/master/zap/controlObjectConnection.cpp#L132)

