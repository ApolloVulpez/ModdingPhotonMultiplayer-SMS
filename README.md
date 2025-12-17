# Photon Multiplayer - Mod creation rundown/walkthrough 🚀

Welcome to my little info packet on how to send data between clients using Unity Photon. 
This is specifically designed for the game Supermarket Simulator, as that is the game I tested it on; however, it should work for any game that uses Photon as a multiplayer backend, as the procedure will likely be the same.
This is for Unity games built in IL2Cpp, this guide will not work for Mono games

Let's get into it

![Made by ApolloVulpez](https://img.shields.io/badge/Made%20by-ApolloVulpez-blue)
![Version](https://img.shields.io/badge/Version-1.0.0-blue)

## Required Dependencies/references for developing with Photon

#### Dependencies

- **⚡PhotonUnityNetworking.dll**: Main dll for Photon
- **⚡ PhotonRealtime.dll**: contains the networking client and the RaiseEvent function
- **⚡ Photon3Unity3D.dll**: Contains the methods for receiving events, their data and specifically options required for sending data
- **⚡ IL2Cppmscorlib.dll**: Required to register an event handler to middle man Photons event handling for receiving functions.

#### References
- **⚡using Photon.Pun;** - PhotonNetwork main
- **⚡using Photon.Realtime;** - Event sending functions
- **⚡using ExitGames.Client.Photon;** - Event receiving functions


## Basic rundown

To set up a basic communication system with Photon to send most data types from the host to the clients connected only 3 things are required. 

- **An event handler**
- **A function to send data**
- **A function to receive that data**

These can be placed in the same cs file or split up into separate files under the same dll. Doesn't matter

Here is an example event handler setup for receiving messages from the host or other clients


```C#
//Middleman function for Photons event handler
    [HarmonyPatch(typeof(PlayerManager), "Awake")]
    [HarmonyPostfix]
    public static void EventReceiver(PlayerManager __instance)
    {
        // Var for the networking client
        var netcli = PhotonNetwork.NetworkingClient;
        // Failsafe if the networking client is not yet initialised
        if (netcli == null) { return; }
        // Var for any events coming through Photon's event handler so that we can pass them on like normal
        var orig = netcli.EventReceived;

        // Our middleman handler, when Photon receives an event, 
        // this runs our event handler while also calling the original event handler. 
        // This allows us to allow Photons' normal events to flow past, while also simultaneously 
        // listening in for any events we actually want to react to.
        // The eventdata is a lambda that runs when an event is called. This is what is allowing all events to flow using ?. If it's not null. 
        // It also runs our event handler to basically check if the event is relevant to us. As you'll see later
           var _photonHandler =
               Il2CppInterop.Runtime.DelegateSupport
                    .ConvertDelegate<Il2CppSystem.Action<EventData>>(
                         (EventData e) =>
                         {
                               orig?.Invoke(e);
                               OnEventHandler(e);
                         });

        netcli.EventReceived = _photonHandler;


    }
```

#### Now that we have our event handler in place

We can now use it to listen to events we actually want to read and react to. 
Here is a basic receiver function that logs a message to the console when an event is received that matches event data code 50

Data codes can be any integer as we'll learn about later 

```c#
    private static void OnEventHandler(EventData eventData)
    {
        //Check event data code to see if its the event we're interested in.
        if (eventData.Code != EVENT_TEST_MESSAGE)
            return;

        // Create an IL2Cpp object to store the incoming data
        Il2CppSystem.Object data = eventData.CustomData;

        //Convert the IL2Cpp object to a string if not null
        string msg = data != null ? data.ToString() : null;

        //Log the string "msg" to the console
        Log.LogInfo($"[Photon] {msg}");
    }
```

This will filter out all events that do not match our data code. Then convert the incoming eventData to a string and print it to the console, simple!
The if statement is a little redundant; I only placed it there as a method to display if something went wrong with the eventdata.

### **Important**: You can use one function to catch multiple events separated by if statements, but it may be worth it to split each event into separate functions
Example: 

```c#
    private static void OnEventHandler(EventData eventData)
    {
        if (eventData.Code == 31)
        {
            //do stuff with this event data
            
        }
        else if (eventData.Code == 2)
        {
            //Do some other stuff with this event data
        }
        else if(eventData.Code == 55)
        {
            //You get the idea
        }
```



Right, let's get to sending some data, shall we? 

Now it's important to note **This can go both ways, from clients and hosts.** So separating between hosts and clients may be required depending on the data! There are a few methods to do this; I'll cover one at the end of this info packet.

Basic send function for most data types: 

```c#
    // Hook may differ depending on the info being sent,
    // in this case, a string when the player hits the O key
    [HarmonyPatch(typeof(PlayerInteraction), "Update")]
    [HarmonyPostfix]
    public static void SendMessage_Host(PlayerInteraction __instance)
    {
        // Checks if the player is the host or a client player
        if (!isLocalPlayer)
            return;
        //Checks if O was pressed
        if (Input.GetKeyInt(KeyCode.O) && Event.current.type == EventType.KeyDown)
        {
            // Fires an event on the PhotonNetwork with the options as follows:
            // Event Data Code, The data (In this case, a string), Event options (This being who should receive it, everyone but the sender), 
            // and finally, how it should be sent (In this case, SendReliable means photon will ensure the data arrives at the client in order with no dropped packets)
            PhotonNetwork.RaiseEvent(
                31,
                "Message received!",
                new RaiseEventOptions { Receivers = ReceiverGroup.Others },
                SendOptions.SendReliable
            );
            // A simple visual log to notify that the event has been fired on the Photon Network
            Log.LogInfo("[Photon] Test event sent");
        }
    }
}
```
### So this is just a simple function that sends a string to all listening clients with the ID 31 and the equivalent OnEventHandler function, which then that function can unpack the string and print it!

Now, with this function, any data type can be sent (at least from my own tests) 

This includes:
- Strings
- Integers
- Float values
- Object References
- Json Serialised Object lists

> Small tip: A method to allow communication strictly between the host and a specific client - You can use "UserID" in "PlayerInstance" (This is specific to SMS, but may exist in other Unity games), then use the client UserID as a reply event code and receiving event code when the host sends data 

A plain text would be something like

- Client: Hey, host, I require the JSON data for the current racks in the building. Can you send it? My userID is 444321 
(Begins listening for events that use ID 44321)
- Host: Of course (Fires event with rack JSON data using event data ID 444321)
- Client: (Picks up on the event fired from the host and now has the data it required)

None of the other clients should pick up on this exchange because their userIDs should be different :)
>Take this with a grain of salt; of course, this will be a lot more complicated in an actual code block. This is only an example, as it's outside of scope for this info packet

# Small tutorial for a basic way to determine if the player is the host or a client.

Create a public bool and then patch the PlayerManager with Harmony on Awake and query for IsMasterClient on LocalPlayer, then set a bool either to true or false depending on the outcome of the IsMasterClient boolean

Code block:
```c#
    public static bool isLocalPlayer { get; set; }
    [HarmonyPatch(typeof(PlayerManager), "Awake")]
    [HarmonyPostfix]
    public static void IsHost(PlayerManager __instance)
    {
        bool flag = __instance.LocalPlayer.IsMasterClient;
        if (flag) { isLocalPlayer = true; }
    }
```
This will allow you to run code based on that public bool on whether its the host client running code or a remote player.


Any questions, follow my card link for contact info

[ambientskai.carrd.co](https://ambientskai.carrd.co/)

