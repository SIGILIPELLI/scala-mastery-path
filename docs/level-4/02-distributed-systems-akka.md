# 02 · Distributed Systems with Akka

Level 3's actors all lived inside one `ActorSystem` in one JVM. **Akka
Cluster** extends the same actor model across multiple JVMs/machines — a
group of `ActorSystem`s that discover each other, track membership, and
route messages between actors regardless of which physical node they're
on. This module covers cluster membership, the mechanism every distributed
Akka feature is built on top of.

```scala title="build.sbt"
scalaVersion := "3.3.1"
libraryDependencies += "com.typesafe.akka" %% "akka-cluster-typed" % "2.8.5"
```

```hocon title="src/main/resources/application.conf"
akka {
  actor.provider = cluster
  remote.artery {
    canonical.hostname = "127.0.0.1"
    canonical.port = 0
  }
  cluster {
    seed-nodes = []
    downing-provider-class = "akka.cluster.sbr.SplitBrainResolverProvider"
  }
}
```

Switching `actor.provider` from the default `local` to `cluster` is what
turns a plain `ActorSystem` into a cluster-capable node — everything else
about defining actors and sending messages (Level 3) stays identical.

## Joining a cluster

```scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.cluster.typed.{Cluster, Join}

val system = ActorSystem(Behaviors.empty[Nothing], "cluster-demo")
val cluster = Cluster(system)
cluster.manager ! Join(cluster.selfMember.address)
Thread.sleep(1000)   // membership is eventual, not instant

println(s"self member status: ${cluster.selfMember.status}")   // self member status: Up
println(s"cluster state members: ${cluster.state.members.size}")  // cluster state members: 1
```

Running one node and joining itself produces a one-member cluster — the
smallest possible one, but the same mechanism scales to hundreds of nodes.
`cluster.selfMember.status` moves through a lifecycle: `Joining` →
`WeaklyUp`/`Up` once the cluster confirms membership, and eventually
`Leaving` → `Exiting` → `Removed` on graceful shutdown.

### The trap: membership changes are eventual, not instant

`cluster.manager ! Join(...)` sends a message and returns immediately —
the node isn't a confirmed cluster member the instant that line runs. Code
that needs to *react* to membership changes should subscribe to cluster
events (`Cluster(system).subscriptions ! Subscribe(...)`) rather than
polling `cluster.state` right after joining, exactly the same "don't
assume async work finished" trap from `Future`/actor `!` in earlier
modules — just at the scale of an entire node joining a cluster instead of
one message being processed.

## Multi-node clusters in practice

A real cluster runs each node as a separate JVM process, each with its own
`canonical.port` and a shared list of `seed-nodes` (well-known addresses
new nodes contact to discover the rest of the cluster):

```hocon
akka.cluster.seed-nodes = [
  "akka://cluster-demo@127.0.0.1:25251",
  "akka://cluster-demo@127.0.0.1:25252"
]
```

Starting two processes with this config (one bound to port `25251`, one to
`25252`) produces a real two-node cluster where each node's `cluster.state.members`
converges to size `2` once gossip propagates. This is naturally beyond
what one runnable code sample in a single process can demonstrate, but the
one-node example above uses the exact same API — `Cluster(system)`, `Join`,
`cluster.state` — that multi-node deployments do.

## Cluster Sharding: the payoff

The reason to reach for a cluster instead of a single beefy `ActorSystem`
is usually **Cluster Sharding** — distributing many stateful actors (one
per user, one per device, one per order) evenly across nodes, with Akka
automatically routing a message for a given entity id to whichever node
currently hosts it, and re-homing it elsewhere if that node fails. Every
piece from Level 3 (a `Behavior[Command]`, state via recursive behaviors)
is reused unchanged — sharding only changes *how a message finds* the right
actor instance, not how that actor is written.

### The trap: distribution isn't free

Cluster Sharding and remote actor communication trade the single-JVM
guarantee of "one at a time, no races" for network calls, serialization,
and the possibility of partial failure (a node can disappear mid-message).
Reach for a cluster only once a single node's actors genuinely can't handle
the throughput or need to survive a node crash — plain Level 3 actors
inside one `ActorSystem` remain the right default for anything that fits on
one machine.

## Cheat sheet

| Need to... | Use |
|---|---|
| Enable clustering for an `ActorSystem` | `akka.actor.provider = cluster` in config |
| Join a cluster | `Cluster(system).manager ! Join(address)` |
| Check this node's status | `cluster.selfMember.status` |
| See all known members | `cluster.state.members` |
| React to membership changes | subscribe to cluster events, don't poll |
| Distribute many stateful actors across nodes | Cluster Sharding (built on the mechanism above) |

## Exercise

Configure a second `application.conf` profile (or pass `-Dakka.remote.artery.canonical.port=25252`
on the command line) and start two separate `sbt run` processes with
matching `seed-nodes` pointing at each other's ports. Confirm both
processes eventually report `cluster.state.members.size == 2`. Then
subscribe to `akka.cluster.ClusterEvent.MemberEvent` on one node (via
`cluster.subscriptions ! Subscribe(listenerActor, classOf[MemberEvent])`)
and print a line whenever the other node joins or leaves, instead of
polling `cluster.state` on a timer.
