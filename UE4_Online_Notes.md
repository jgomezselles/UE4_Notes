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
To launch the game in multiple windows, spawning different clients, just set up the number of `Players` under multiplayer options.



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


