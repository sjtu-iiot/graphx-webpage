Pregel and Shortest Path Algorithm
==================================

Graphs are inherently recursive data structures. Many important graph algorithms rely on **iterative operations**. However, during each iteration, programers have to `cache` the new results manually, which is inconvenient and uncontrollable. To solve this problem and make the iteration easier to use, GraphX provides a Pregel-like API.

 At a high level the Pregel operator in GraphX is a **Pregel-like bulk-synchronous** parallel messaging abstraction. The Pregel operator executes in a series of super steps in which vertices receive the sum of their inbound messages from the previous super step, compute a new value for the vertex property, and then send messages to neighboring vertices in the next super step.

The following is the type signature of the Pregel operator as well as a sketch of its implementation:

```scala
def pregel[A]
    (initialMsg: A,
     maxIter: Int = Int.MaxValue,
     activeDir: EdgeDirection = EdgeDirection.Out)
    (vprog: (VertexId, VD, A) => VD,
     sendMsg: EdgeTriplet[VD, ED] => Iterator[(VertexId, A)],
     mergeMsg: (A, A) => A)
  : Graph[VD, ED] = {...}
```

### Parameter interpretation

* `A`: the Pregel message type
* `initialMsg`: the message each vertex will receive at the first iteration
* `maxIter`: maxIter the maximum number of iterations to run for
* `activeDir`:the direction of edges incident to a vertex that received a message in the previous round on which to run sendMsg.
* `vprog`: the user-defined vertex program which runs on each vertex and receives the inbound message and computes a new vertex value.
* `sendMsg`:a user supplied function that is applied to out edges of vertices that received messages in the current iteration
* `mergeMsg`:a user supplied function that takes two incoming messages of type A and merges them into a single message of type A.

The complete Pregel API is provided in [Pregel.scala][].
[Pregel.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/Pregel.scala

Pregel will begin with sending the initialMsg to each vertice in graph and the vertex program is invoked on all vertices. The code is as follows:

```
var g = mapVertices( (vid, vdata) => vprog(vid, vdata, initialMsg) ).cache()
```


### The loop process

1. Receive the messages. Vertices that didn't get any messages do not appear in newVerts.
2. Update the graph with the new vertices.
3. Send new messages.
4. Materializes `messages`, `newVerts`, and the vertices of `g`. This hides oldMessages (depended on by newVerts), newVerts (depended on by messages), and the vertices of prevG (depended on by newVerts, oldMessages, and the vertices of g)
5. Unpersist the RDDs hidden by newly-materialized RDDs
6. Count the iteration


Note: For more details, you can refer to [the source code of Pregel][].
[the source code of Pregel]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/Pregel.scala


Example: Single Source Shortest Path algorithm using Pregel
-----------------------------------------------------------

Pregel API can help us calculate PageRank and Shortest Path easily. Here is an example about how to solve the Single Source Shortest Path (SSSP) problem using Pregel.

### Relaxation operation

Till now, computer scientists have proposed several algorithms for the shortest-path problem, such as Dijkstra's algorithm and Bellman–Ford algorithm. These algorithms have different implementations, **but the cores of them are the same, the relaxation operation**.

Note: The relaxation operation. If distance from _a_ to _c_ `dis[a][c]` is longer than the distance from _a_ to _b_ `dis[a][b]` plus that from _b_ to _c_ `dis[b][c]`, then update `dis[a][c] <= dis[a][b] + dis[b][c]`.

### Using Pregel to solve SSSP

Import the graph, define the source vertex, and initialize the distance used to be iterated (we will use Dijkstra’s algorithm):

```scala
scala> import org.apache.spark._
scala> import org.apache.spark.graphx._
scala> import org.apache.spark.rdd.RDD
scala> val graph = GraphLoader.edgeListFile(sc, "hdfs://192.168.17.240:9222/input/yuhc/web-Google/web-Google.txt")
scala> val sourceId: VertexId = 0
scala> val g = graph.mapVertices( (id, _) =>
     |   if (id == sourceId) 0.0
     |   else Double.PositiveInfinity
     | )
```

Note: For the dataset we used, you can reference [Graph Generating and Data Loading](https://snap.stanford.edu/data/web-Google.html).

Then use Pregel API simply by

```scala
scala> val sssp = g.pregel(Double.PositiveInfinity)(
     |   (id, dist, newDist) => math.min(dist, newDist),
     |   triplet => {
     |     if (triplet.srcAttr + triplet.attr < triplet.dstAttr) {
     |       Iterator((triplet.dstId, triplet.srcAttr + triplet.attr))
     |     }
     |     else {
     |       Iterator.empty
     |     }
     |   },
     |   (a, b) => math.min(a, b)
     | )
```

View the result of it:

```scala
scala> sssp.vertices.take(10).mkString("\n")
res0: String =
(354796,11.0)
(672890,11.0)
(129434,11.0)
(194402,15.0)
(199516,8.0)
(332918,13.0)
(170792,11.0)
(386896,11.0)
(691634,8.0)
(291526,Infinity)
```

Shortest-Path Algorithm in GraphX Lib
-------------------------------------

Shortest-Path algorithm has also been provided in [lib/ShortestPaths.scala][]. Instead of calculating the Single Source Shortest Path (SSSP), it computes shortest paths to the given set of landmark vertices, returning a graph where each reachable landmark. And of course, it also invokes Pregel to implement iterative operations.
[lib/ShortestPaths.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/lib/ShortestPaths.scala

* `SPMap` is a map type ` type SPMap = Map[VertexId, Int]` storing the map from the vertex id of a landmark to the distance to that landmark.
* `makeMap` is a mapping function: `private def makeMap(x: (VertexId, Int)*) = Map(x: _*)`
* `addMaps` chooses the minimal distance value as the `VertexId -> Distance` map.
* `vertexProgram` calls addMaps and does the same thing.
* `incrementMap` increases the distance of the next hop (as all weights of the edges are considered as 1). The direction of the iteration is a bit strange–it jumps from the destination vertex to the source vertex on a edge. But of course it doesn’t affect the result.

```scala
object ShortestPaths {
  ...
  def run[VD, ED: ClassTag](graph: Graph[VD, ED], landmarks: Seq[VertexId]): Graph[SPMap, ED] = {
    val spGraph = graph.mapVertices { (vid, attr) =>
      if (landmarks.contains(vid)) makeMap(vid -> 0) else makeMap()
    }

    val initialMessage = makeMap()

    def vertexProgram(id: VertexId, attr: SPMap, msg: SPMap): SPMap = {
      addMaps(attr, msg)
    }

    def sendMessage(edge: EdgeTriplet[SPMap, _]): Iterator[(VertexId, SPMap)] = {
      val newAttr = incrementMap(edge.dstAttr)
      if (edge.srcAttr != addMaps(newAttr, edge.srcAttr)) Iterator((edge.srcId, newAttr))
      else Iterator.empty
    }

    Pregel(spGraph, initialMessage)(vertexProgram, sendMessage, addMaps)
  }
}
```


Reference
---------

1. [Apache GraphX Official Guide](http://spark.apache.org/docs/latest/graphx-programming-guide.html)
