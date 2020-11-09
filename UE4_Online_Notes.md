# UE4 Multiplayer - Notes
These are notes taken from some sessions held with @Balgy and Udemy's [Unreal Multiplayer Master: Video Game Dev In C++](https://www.udemy.com/course/unrealmultiplayer/) course. These are personal notes, so some things may be inaccurate.

The reader is advised to take a look to the [UE4 Network Compendium](http://cedric-neukirchen.net/Downloads/Compendium/UE4_Network_Compendium_by_Cedric_eXi_Neukirchen.pdf) created by [Cedric 'eXi' Neukirchen](http://cedric-neukirchen.net), which provides a large and comprehensive summary of the most useful assets provided by UE4, and is a perfect support documentation to keep always at hand.

# Playground assets
UE4 Editor provides a starting project which can serve to play around with the concepts explained in this document. It can be created by starting a new project of kind [Third Person](https://docs.unrealengine.com/en-US/Resources/Templates/ThirdPerson/index.html).

## Creating a server
Once the project is in place, a server can be spawned by executing:
```bash
"</path/UE4_Editor_Executable>" "</path/your_project.uproject>" <path_to_map_folder> -server -log
```

## Creating clients
Clients can be spawned with:
```bash
"</path/UE4_Editor_Executable>" "</path/your_project.uproject>" <local_ip_address> -game -log
```
> Official [doc](https://docs.unrealengine.com/en-US/Gameplay/Networking/Server/index.html)!

## In the editor
To launch the game in multiple windows, spawning different clients, just set up the number of `Players` under multiplayer options, and select Net Mode: `Play as listen server`.



# Multiplayer models
When doing a multiplayer game, one of the main things that need to be taken into account is that `Objects` have a `State` (Position, Health...), which might change when are subject to some `Actions` (Interactions, pressing a button...). Both a `State` and an `Action` are evaluated to obtain another `State` after a `Tick`.

## Peer-to-peer model
In a peer-to-peer model, one peer sends its `Action` to all the other peers and, as synchronization is needed, every `Action` needs to wait for confirmation. This model is not really viable because it feeds the network with lots of communications in which bottlenecks are produced, since there's always the need to wait for the slowest peer (for example).

It also encompasses potentially security issues, because messages are not validated in a *central* system.

## Client-Server model
The one Unreal uses. Communication is done via a central unit, the server, which validates communications and states, and is in charge of updating all clients (players).

The server can be run by one player. I does not need to be in a different machine.
> Official [doc](https://docs.unrealengine.com/en-US/Gameplay/Networking/Server/index.html).

# Authority & Replication
To detect, distinguish or decide code to be run in the server and/or in the clients, the function `bool` [AActor::HasAuthority()](https://docs.unrealengine.com/en-US/API/Runtime/Engine/GameFramework/AActor/HasAuthority/index.html), returns `true` if this code is being run on the server and `false` otherwise.

An `AActor` is one of the building blocks to provide replication through the network. To achieve a certain property to be replicated (updated/noticed) across the clients by the server, **both** the `AAcator` and the wanted `UProperty` needs to be set as `Replicated`.

Replication needs to be done in [AActor::BeginPlay()](https://docs.unrealengine.com/en-US/API/Runtime/Engine/GameFramework/AActor/BeginPlay/index.html), not in the constructor, to allow replication being set in the server once all pieces are in place.

For example, to replicate movement for all actors of type `AMyActor`:

```cpp
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority()) // Run the following code only on server!
    {
        SetReplicates(true); //Set actor replication
        SetReplicateMovement(true); //Set movement replication
    }
}
```
So, after setting replication properly, to get this actor's movement to be processed by the server, and *updated* to the clients, the `AMyActor` class will implement the `Tick()` function as:

```cpp
void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    if (HasAuthority())
    {
        FVector Location = GetActorLocation();
        Location += FVector(Speed * DeltaTime, 0, 0);
        SetActorLocation(Location);
    }
}
```
But if, for example we had implemented this to be run only in the client (i.e. `if(!HasAuthority)`), we would see in the client the object instantiation of `AMyActor` moving **but**, as the server is **authoritative**, we would collide with the `AMyActor` object even if we don't see it (we would be seeing it in a different place after the movement!). This is because our movement is *validated* in the server, and the `AMyActor`object has not moved as to the server's *eyes*.

> Official [doc](https://docs.unrealengine.com/en-US/Gameplay/Networking/Actors/Components/index.html).

## Replicating States

Replication in UE4 is done via [RPCs](https://docs.unrealengine.com/en-US/Gameplay/Networking/Actors/RPCs/index.html), which are functions called locally, but executed remotely. To define a function as an RPC, use one of the following flavors, depending on your needs:

```cpp
UFUNCTION(Client) //Called in server, executed in clients
void ClientRPCFunction;

UFUNCTION(Server) //Called in clients, executed in server
void ServerRPCFunction;

UFUNCTION(NetMulticast) //Called in server or clients, executed in server and clients
void MulticastRPCFunction;
```
> As a convention, RPC functions shall start with *Client*, *Server*, or *Multicast*, prefix.

## Reliability

Reliability is a way for UE4 of stablishing priorities on networking messages, for example, to avoid/act-on network latency/jamming. Reliable RPCs are re-sent if they are lost, which can potentially add delays. Reliable and unreliable packets may (or may not) be merged in the same frame, and there's no guarantee of ordering in reliable vs non-reliable calls.

To enforce RPCs are executed on a remote machine, (for instance, retried if lost) reliability shall be specified:

```cpp
UFUNCTION(Client, Reliable)
void ClientRPCFunction;
```

## Validation

In order to avoid cheating, or other possible not-controlled issues, client->server RPCs are enforced to be validated, and only executed in the server. To achieve this, `WithValidation` will force the implementation (and execution) of two *auxiliary* functions: `bool SomeRPCFunction_Validate` and `SomeRPCFunction_Implementation`. Validation function will result in caller disconnection upon failure (returning false) and Implementation will be called otherwise.

## Actor Roles

*Roles* are assigned to Actors depending on where are they being run and the authority of the machine running them.

For instance, let's think of an [NPC](https://en.wikipedia.org/wiki/Non-player_character) Actor running on a server. It will have a Role of type `ROLE_Authority` and a RemoteRole of type  `ROLE_SimulatedProxy`. The same Actor, running in the client, will have a Role of type `ROLE_SimulatedProxy` and a `ROLE_Authority` RemoteRole.

As the actual processing of the Actor state is done in the server, in order to avoid glitches and *laggy* game experience, the Actor is *simulated* on the client side, between every update. 

It is up to the particular replication implementation whether if the simulated state is *benchmarked* (and possibly updated) against the server or not. Custom implementations and extrapolation of states between server updates is allowed.

When talking about Actors possessed by a `PlayerController`, the Role on the client is `ROLE_AutonomousProxy`. In this particular Role, the input from the player is taken as information to help the simulation on the client side.

> Role can be checked by using `GetLocalRole()` function.

## Property Replication

When we talk about *Actor Replication*, it is actually a set of properties what is being replicated. It is up to the implementation to decide the properties that are important to be [replicated](https://docs.unrealengine.com/en-US/Gameplay/Networking/Actors/Properties/index.html).

There are different ways of defining and replicating a property. In the next example, `NotSimulatedProperty` is only computed in the server, and updated to the clients. `SimulatedProperty` is computed in the server AND in the client, but it gets updated every certain time. In this example, `SimulatedProperty` is also registering a callback which triggers some code every time an update is received due to replication.

1) Define the property (class variable) in your header file:

    ```cpp
    UPROPERTY(Replicated)
    float NotSimulatedProperty;

    UPROPERTY(ReplicatedUsing=OnRep_SimulatedProperty)
    float SimulatedProperty

    //'OnRep_' prefix is predefined in UE4.
    //The following function is run every time SimulatedProperty
    //receives an update.
    UFUNCTION()
    void OnRep_SimulatedProperty();
    ```
2) Inform that this Actor's properties will be replicated:
    ```cpp
    #include "Net/UnrealNetwork.h"
    AMyActor::AMyActor()
    {
        /*
        * Some constructor code here
        */
        bReplicates = true; //Inherited from AActor
    }
    ```
3) Override lifetime replicated properties:
    ```cpp
    void AMyActor::GetLifetimeReplicatedProps(TArray< FLifetimeProperty > & OutLifetimeProps) const
    {
        Super::GetLifetimeReplicatedProps(OutLifetimeProps);
        DOREPLIFETIME(AMyActor, NotSimulatedProperty);
        DOREPLIFETIME(AMyActor, SimulatedProperty);
    }
    ```
4) Write the different logic for the server and client:
    ```cpp
    void AMyActor::Tick(float DeltaTime)
    {
        //This is the actual difference between simulating or not.
        //If the code is run only on the server, the client will
        //only show updates. If it's run in both sides, it is calcluated
        //on the client, and then updated if needed.
        SimulatedProperty = CalculateNewPropertyValue();
        if (HasAuthority()) 
        {
            SomeProperty = UpdateNotSimulatedProperty();
        }
    }
    ```
4) Implement On_Rep callback:
    ```cpp
    void AMyActor::OnRep_SimulatedProperty();
    {
        /*
        * Do something here about the new value. It has actually
        * been changed. But you might want to reflect it somewhere.
        * For example, if the variable is the Actor's Health, you may want
        * to trigger some action.
        * Another option is that we could try to check if the new value is
        * very different, and we might want to, instead of just replacing
        * it, do something to smoothen the transition.
        */
    }
    ```

## Update Frequency

[Actors](https://docs.unrealengine.com/en-US/API/Runtime/Engine/GameFramework/AActor/index.html) are not being replicated for every Tick or update, because that would imply a huge bandwidth overload. Instead, an Actor is updated at a rate (times per second) defined in `AActor::NetUpdateFrequency`. Set this variable in `BeginPlay()` function.

```cpp
void AMyActor::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        NetUpdateFrequency = MyDesiredRate; //Number of replications per second
    }
}
```
## Replicating with Structs

As we might want to replicate a certain set of variables, it is normally a good idea to create a [structure](https://docs.unrealengine.com/en-US/Programming/UnrealArchitecture/Reference/Structs/index.html) (`USTRUCT`) containing related variables which, in the end, represent the state of an Actor.

Such structures may be used to help simulating and correcting possibly lost or huge changes in updates for the state.

# Simulating lag
Check out this [blog post](https://www.unrealengine.com/en-US/blog/finding-network-based-exploits) to make some tests on how your code reacts to different networking issues such as packet losses, lag, packets order and packet duplications.