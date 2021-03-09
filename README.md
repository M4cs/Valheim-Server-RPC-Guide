# Valheim Server Validated RPC Guide
A Guide for making a powerful RPC request/message system with Valheim's ZRoutedRPC

# Introduction
Valheim comes packed with a robust RPC messaging system through the ZRoutedRPC class. You can patch the RPC_* methods provided but you can also create your own RPC methods. Before you start, it may help to break your project up into a server side project and a client side project. This will make your RPC methods a lot easier to work with because you know where you will be handling them, and you can create server specific, and client specific RPC handlers. 

The RPC messaging system Valheim uses thrives off ZPackages. These are serializable data packets that can be sent over the RPC system. You can write almost all standard types of data to them including `bool, byte, byte[], char, double, int, long, Quaternion, sbyte, float, string, uint, ulong, Vector2i, Vector3, and ZDOID`. While `Quaternion, Vector2i, and Vector3` are Unity types, you can still de/serialize them with ZPackages. 

# Requirements

* BepInEx >=5.4
* HarmonyLib (Ships w/ BepInEx)
* Publicized Assemblies (For our example)

# Creating Server Validated RPC Requests

When creating your custom RPC methods, you can name them whatever you want, however it's recommended you prefix them with `RPC_` as that is what the rest of the codebase uses and it's common practice. It also helps to distinguish your RPC methods from others. It's recommended that you create either an RPC class or an RPC namespace inside of your Mod's namespace. You can do this by creating a folder called `RPC` or simply creating a new class with the name you desire. For the examples, our RPC methods will be in the `Mod.RPC` namespace, inside of the `RPC` class.

## Where can this be used?

This guide was written with the idea that you are hosting a **dedicated** server and providing a Server and Client mod. One to run on the server, one to run client side. **I simply don't know how this will work P2P, you will probably need to add the server functions to the Client mod and check for `ZNet.instance.IsServer()` if you are the host or not.**

## The Messaging Flow

<p align="center">
  <a><img width="450" height="700" src="https://mbcdn.sfo2.digitaloceanspaces.com/RPCDiagram.png"/></a>
<p>
  
1. The client makes a request to the server with the information necessary for the request.
2. The server reads the ZPackage sent by the client. It reads the ZPackage, validates the request.

**If Client Request Is Valid:**
1. Sends a new package, or the package sent by the client, to the correct clients.
2. Client will handle this locally.

**If Client Request Is Invalid:**
1. Sends an error event to the client.
2. Client handles the error event.

but what does this look like in code?

## Our Custom Handlers

**The usecase:** I want to create a function, that allows clients to run a chat command as admin, and send a server announcement. It will display in the client's chat without using the RPC chat (you could replace this with logic to do something else, this is purely an example).

First, we need to create an RPC function for our server to handle our Client request. In the `RPC` class of our **Server** project, we create a function called `RPC_RequestServerAnnouncement`. This will take a sender ID and a ZPackage. We assume that the ZPackage holds a single string with the message the Client wants to send. You will see us build this out later.

```cs
public static void RPC_RequestServerAnnouncement(long sender, ZPackage pkg)
{
    if (pkg != null && pkg.Size() > 0)
    { // Check that our Package is not null, and if it isn't check that it isn't empty.
        ZNetPeer peer = ZNet.instance.GetPeer(sender); // Get the Peer from the sender, to later check the SteamID against our Adminlist.
        if (peer != null)
        { // Confirm the peer exists
            string peerSteamID = ((ZSteamSocket)peer.m_socket).GetPeerID().m_SteamID.ToString(); // Get the SteamID from peer.
            if (
                ZNet.instance.m_adminList != null &&
                ZNet.instance.m_adminList.Contains(peerSteamID)
            )
            { // Check that the SteamID is in our Admin List.
                string msg = pkg.ReadString(); // Read the message from the user.
                if (!msg.Equals("I hate you"))
                { // Example of validating the string.
                    pkg.SetPos(0); // Reset the position of our cursor so the client's can re-read the package.
                    ZRoutedRpc.instance.InvokeRoutedRPC(0L, "EventServerAnnouncement", new object[] { pkg }); // Send our Event to all Clients. 0L specifies that it will be sent to everybody
                }
                else
                {
                    ZPackage newPkg = new ZPackage(); // Create a new ZPackage.
                    newPkg.Write("That's not nice!"); // Shame them.
                    ZRoutedRpc.instance.InvokeRoutedRPC(sender, "BadRequestMsg", new object[] { newPkg }); // Send the error message.
                }
            }
        } else {
            ZPackage newPkg = new ZPackage(); // Create a new ZPackage.
            newPkg.Write("You aren't an Admin!"); // Tell them what's going on.
            ZRoutedRpc.instance.InvokeRoutedRPC(sender, "BadRequestMsg", new object[] { newPkg }); // Send the error message.
        }
    }
}
```

This will give us a basic server validator and allow us to send the events to the client whether it's completing the request, or sending them an error. `0L` as our target makes our RPC message send to all clients. However! Since we call `0L` for the target from our server, we need to create a Mock client handler function for the server, but we will just make that an empty return. For consistency we will name this the same as our Client RPC function that we will make next! Add a function in the same **Server** `RPC` class and name it `RPC_EventServerAnnouncement`.

```cs
public static void RPC_EventServerAnnouncement(long sender, ZPackage pkg) {
    return;
}
```

Now we can create our Client side handler. This will be the same name as the function above, except this is where we put our Client logic! Make a new function in the `RPC` class of your **Client** mod.

```cs
public static void RPC_EventServerAnnouncement(long sender, ZPackage pkg) {
    if (sender == ZRoutedRpc.instance.GetServerPeerID() && pkg != null && pkg.Size() > 0) { // Confirm our Server is sending the RPC
        string announcement = pkg.ReadString();
        if (announcement != "") { // Make sure it isn't empty
            Chat.instance.AddString("Server", announcement, Talker.Type.Shout); // Add our server announcement to the Client's chat instance
        }
    }
}
```

We now have to create a Client handler for error messages. I like to display an error in chat, but you can do whatever you want to! Add a new function called `RPC_BadRequestMsg`.
```cs
public static void RPC_BadRequestMsg(long sender, ZPackage pkg) {
    if (sender == ZRoutedRpc.instance.GetServerPeerID() && pkg != null && pkg.Size() > 0) { // Confirm our Server is sending the RPC
        string msg = pkg.ReadString(); // Get Our Msg
        if (msg != "") { // Make sure it isn't empty
            Chat.instance.AddString("Server", "<color=\"red\">" + msg + "</color>", Talker.Type.Normal); // Add to chat with red color because it's an error
        }
    }
}
```

Now we have to create a Mock server handler for the request. Add an empty function with the same name as your Server's request handler.

```cs
public static void RPC_RequestServerAnnouncement(long sender, ZPackage pkg) {
    return;
}
```

As far as creating the logic for our custom RPC system, we are finished! We just have to patch the `Game` class, to register our new RPC functions. In **both the Client and Server projects**, create a patch for Game.Start() and register your functions.

For the **Server** project, we can patch all the functions we created.

```cs
[HarmonyPatch(typeof(Game), "Start")]
public static class GameStartPatch {
    private static void Prefix() {
        ZRoutedRpc.instance.Register("RequestServerAnnouncement", new Action<long, ZPackage>(RPC.RPC_RequestServerAnnouncement); // Our Server Handler
        ZRoutedRpc.instance.Register("EventServerAnnouncement", new Action<long, ZPackage>(RPC.RPC_EventServerAnnouncement); // Our Mock Client Function
    }
}
```

For the **Client** project, we can patch all the functions we created.

```cs
[HarmonyPatch(typeof(Game), "Start")]
public static class GameStartPatch {
    private static void Prefix() {
        ZRoutedRpc.instance.Register("RequestServerAnnouncement", new Action<long, ZPackage>(RPC.RPC_RequestServerAnnouncement); // Our Mock Server Handler
        ZRoutedRpc.instance.Register("EventServerAnnouncement", new Action<long, ZPackage>(RPC.RPC_EventServerAnnouncement); // Our Client Function
        ZRoutedRpc.instance.Register("BadRequestMsg", new Action<long, ZPackage>(RPC.RPC_BadRequestMsg); // Our Error Handler
    }
}
```

That's it! We are all setup to use our new custom RPC methods with server validation! If you'd like to initiate the request, we'd patch the Chat class and handle a custom command on our **Client** project.

```cs

[HarmonyPatch(typeof(Chat), "InputText")]
public static class ChatHandler
{
    private static bool Prefix(ref Chat __instance)
    {
        string text = __instance.m_input.text; // Get the chat text
        string[] textSplit = text.Split(' '); // Split up args
        if (textSplit.Length > 1) { // Make sure it's more than 1 word
            if (textSplit[0] == "/announce") { // Check if it's our command
                ZPackage pkg = new ZPackage(); // Create ZPackage
                string msg = ""; // Make msg
                for (int i = 1; i < textSplit.Length; i++) {
                    msg += textSplit[i] + " ";
                }
                pkg.Write(msg); // Write msg to ZPackage
                
                // Send msg over RPC to server
                ZRoutedRpc.instance.InvokeRoutedRpc(ZRoutedRpc.instance.GetServerPeerID(), "RequestServerAnnouncement", new object[] { pkg });
                return false; // Skip original operation
            }
        }
        return true;
    }
}
```

This will start our RPC flow with the server and if the validation works, you will see the announcement, or you will get an error because you aren't an admin or said you hate everybody 😜

### Need help?

Come check out the Valheim Modding [discord](https://discord.gg/KEHFbeyv52). We have a great community that is always willing to help out! You can find me in there with the username @macs 
