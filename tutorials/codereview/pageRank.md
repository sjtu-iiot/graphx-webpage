PageRank in GraphX
==================

What is PageRank
----------------

`PageRank` is an algorithm used by Google Search to rank websites in their search engine results [1]. `PageRank` is a way of measuring the importance of website pages.

![PageRank_img](http://upload.wikimedia.org/wikipedia/commons/thumb/6/69/PageRank-hi-res.png/320px-PageRank-hi-res.png)

`PageRank` works by counting the number and quality of links to a page to determine a rough estimate of how important the website is. The underlying assumption is that more important websites are likely to receive more links from other websites.


PageRank in GraphX
------------------

`PageRank` measures the importance of each vertex in a graph, assuming an edge from `u` to `v` represents an endorsement of `v`’s importance by `u`. For example, if a Twitter user is followed by many others, the user will be ranked highly.

GraphX comes with static and dynamic implementations of `PageRank` as methods on the `PageRank object`. `Static PageRank` runs for a fixed number of iterations, while `dynamic PageRank` runs until the ranks converge (i.e., stop changing by more than a specified tolerance). `GraphOps` allows calling these algorithms directly as methods on `Graph`.


Using Pregel API to Calculate PageRank
--------------------------------------

GraphX provides `PageRank` API for users to calculate the `PageRank` of a graph conveniently. The API is defined in [PageRank.scala]. Let’s review the `Pregel` API (defined in [Pregel.scala]) and learn how to calculate `PageRank` using `Pregel` API.

### Import the libraries

```scala
scala> import org.apache.spark._
scala> import org.apache.spark.graphx._
scala> import org.apache.spark.rdd.RDD
```

### Load the edges

```scala
scala> val graph = GraphLoader.edgeListFile(sc, "hdfs://192.168.17.240:9222/input/yuhc/web-Google/web-Google.txt")
```

### Associate the out-degree with each vertex using `outerJoinVertices` by

```scala
scala> val tmp = graph.outerJoinVertices(graph.outDegrees) {
     |   (vid, vdata, deg) => deg.getOrElse(0)
     | }
scala> tmp.vertices.take(10)
res0: Array[(org.apache.spark.graphx.VertexId, Int)] = Array((354796,1), (672890,0), (129434,2), (194402,1), (199516,20), (332918,3), (170792,9), (386896,18), (691634,11), (291526,7))
```

### Set the weight on the edges based on the degree using `mapTriplets` by

```scala
scala> val edgetmp = tmp.mapTriplets( e => 1.0/e.srcAttr )
scala> edgetmp.triplets.take(5)
res1: Array[org.apache.spark.graphx.EdgeTriplet[Int,Double]] = Array(((0,4),(11342,14),0.25), ((0,4),(824020,11),0.25), ((0,4),(867923,12),0.25), ((0,4),(891835,10),0.25), ((1,10),(53051,0),0.1))
```

### Also set the vertex attributes to the initial PageRank values

```scala
scala> val initialGraph = edgetmp.mapVertices( (id, attr) => 1.0 )
scala> initialGraph.vertices.take(10)
res2: Array[(org.apache.spark.graphx.VertexId, Double)] = Array((354796,1.0), (672890,1.0), (129434,1.0), (194402,1.0), (199516,1.0), (332918,1.0), (170792,1.0), (386896,1.0), (691634,1.0), (291526,1.0))
```

Now the vertices in `initialGraph` are assigned initial PageRank `1.0`, and the edges in `initialGraph` store the out-degree information.

### Combine all these operations into an one-line code

```scala
val initialGraph: Graph[Double, Double] = graph
  .outerJoinVertices(graph.outDegrees) {
    (vid, vdata, deg) => deg.getOrElse(0)
  }
  .mapTriplets(e => 1.0 / e.srcAttr)
  .mapVertices((id, attr) => 1.0)
```

Assume `val initialMessage = 0.0`, the number of iterations `val numIter = 100` (you can take a smaller value), and the damping factor `val resetProb = 0.15`. Other message handlers are programmed according to the PageRank algorithm:

```scala
def vertexProgram(id: VertexId, attr: Double, msgSum: Double): Double =
  resetProb + (1.0 - resetProb) * msgSum
def sendMessage(edge: EdgeTriplet[Double, Double]): Iterator[(VertexId, Double)] =
  Iterator((edge.dstId, edge.srcAttr * edge.attr))
def messageCombiner(a: Double, b: Double): Double = a + b
```

### Finally call the `Pregel` API

```scala
val pagerankGraph = Pregel(initialGraph, initialMessage, numIter)(
  vertexProgram, sendMessage, messageCombiner)
```

### Watch the result

```scala
scala> initialGraph.vertices.take(10)
res3: Array[(org.apache.spark.graphx.VertexId, Double)] = Array((354796,0.19213500326152516), (672890,0.2908035889020495), (129434,0.3887609101479692), (194402,0.6070857155891107), (199516,1.4700031805071416), (332918,0.3013570799851124), (170792,0.2239911438748651), (386896,0.2885166353065909), (691634,3.186713734863332), (291526,0.6765487457543149))
```

Simply using PageRank API
-------------------------

As mentioned at the before, users can calculate `PageRank` simply using `PageRank` API defined in [PageRank.scala]. This API is wrapped in [GraphOps.scala]:

```scala
def pageRank(tol: Double, resetProb: Double = 0.15): Graph[Double, Double] = {
  PageRank.runUntilConvergence(graph, tol, resetProb)
}
```

`tol` is the tolerance of the error. The smaller the `tol` is, the more accurate the result will be.

### Using PageRank API

```scala
scala> val rank = graph.pageRank(0.01).vertices.take(10)
rank: Array[(org.apache.spark.graphx.VertexId, Double)] = Array((354796,0.18810908106478785), (672890,0.27422939743907243), (129434,0.3664605076158495), (194402,0.5874897431581598), (199516,1.3022385827251726), (332918,0.2847157589683162), (170792,0.21475148570654695), (386896,0.2767245351551102), (691634,2.803096788390362), (291526,0.6515237426896987))
```

You can also use `staticPageRank` to calculate a specific times of iteration.

```scala
def staticPageRank(numIter: Int, resetProb: Double = 0.15): Graph[Double, Double] = {
  PageRank.run(graph, numIter, resetProb)
}
```

[PageRank.scala] implements two implementations of `PageRank`. The first implementation uses the standalone `Graph` interface and runs `PageRank` for a fixed number of iterations; The second implementation uses the `Pregel` interface and runs `PageRank` until convergence (just like our example).


[PageRank.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/lib/PageRank.scala
[Pregel.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/Pregel.scala
[GraphOps.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/GraphOps.scala


Source Code Analysis
--------------------

The codes of `PageRank` are shown as follows.

```scala
object PageRank extends Logging {

  /**
   * Run PageRank for a fixed number of iterations returning a graph
   * with vertex attributes containing the PageRank and edge
   * attributes the normalized edge weight.
   *
   * @tparam VD the original vertex attribute (not used)
   * @tparam ED the original edge attribute (not used)
   *
   * @param graph the graph on which to compute PageRank
   * @param numIter the number of iterations of PageRank to run
   * @param resetProb the random reset probability (alpha)
   *
   * @return the graph containing with each vertex containing the PageRank and each edge
   *         containing the normalized weight.
   *
   */
  def run[VD: ClassTag, ED: ClassTag](
      graph: Graph[VD, ED], numIter: Int, resetProb: Double = 0.15): Graph[Double, Double] =
  {
    // Initialize the PageRank graph with each edge attribute having
    // weight 1/outDegree and each vertex with attribute 1.0.
    var rankGraph: Graph[Double, Double] = graph
      // Associate the degree with each vertex
      .outerJoinVertices(graph.outDegrees) { (vid, vdata, deg) => deg.getOrElse(0) }
      // Set the weight on the edges based on the degree
      .mapTriplets( e => 1.0 / e.srcAttr, TripletFields.Src )
      // Set the vertex attributes to the initial pagerank values
      .mapVertices( (id, attr) => resetProb )

    var iteration = 0
    var prevRankGraph: Graph[Double, Double] = null
    while (iteration < numIter) {
      rankGraph.cache()

      // Compute the outgoing rank contributions of each vertex, perform local preaggregation, and
      // do the final aggregation at the receiving vertices. Requires a shuffle for aggregation.
      val rankUpdates = rankGraph.aggregateMessages[Double](
        ctx => ctx.sendToDst(ctx.srcAttr * ctx.attr), _ + _, TripletFields.Src)

      // Apply the final rank updates to get the new ranks, using join to preserve ranks of vertices
      // that didn't receive a message. Requires a shuffle for broadcasting updated ranks to the
      // edge partitions.
      prevRankGraph = rankGraph
      rankGraph = rankGraph.joinVertices(rankUpdates) {
        (id, oldRank, msgSum) => resetProb + (1.0 - resetProb) * msgSum
      }.cache()

      rankGraph.edges.foreachPartition(x => {}) // also materializes rankGraph.vertices
      logInfo(s"PageRank finished iteration $iteration.")
      prevRankGraph.vertices.unpersist(false)
      prevRankGraph.edges.unpersist(false)

      iteration += 1
    }

    rankGraph
  }

  /**
   * Run a dynamic version of PageRank returning a graph with vertex attributes containing the
   * PageRank and edge attributes containing the normalized edge weight.
   *
   * @tparam VD the original vertex attribute (not used)
   * @tparam ED the original edge attribute (not used)
   *
   * @param graph the graph on which to compute PageRank
   * @param tol the tolerance allowed at convergence (smaller => more accurate).
   * @param resetProb the random reset probability (alpha)
   *
   * @return the graph containing with each vertex containing the PageRank and each edge
   *         containing the normalized weight.
   */
  def runUntilConvergence[VD: ClassTag, ED: ClassTag](
      graph: Graph[VD, ED], tol: Double, resetProb: Double = 0.15): Graph[Double, Double] =
  {
    // Initialize the pagerankGraph with each edge attribute
    // having weight 1/outDegree and each vertex with attribute 1.0.
    val pagerankGraph: Graph[(Double, Double), Double] = graph
      // Associate the degree with each vertex
      .outerJoinVertices(graph.outDegrees) {
        (vid, vdata, deg) => deg.getOrElse(0)
      }
      // Set the weight on the edges based on the degree
      .mapTriplets( e => 1.0 / e.srcAttr )
      // Set the vertex attributes to (initalPR, delta = 0)
      .mapVertices( (id, attr) => (0.0, 0.0) )
      .cache()

    // Define the three functions needed to implement PageRank in the GraphX
    // version of Pregel
    def vertexProgram(id: VertexId, attr: (Double, Double), msgSum: Double): (Double, Double) = {
      val (oldPR, lastDelta) = attr
      val newPR = oldPR + (1.0 - resetProb) * msgSum
      (newPR, newPR - oldPR)
    }

    def sendMessage(edge: EdgeTriplet[(Double, Double), Double]) = {
      if (edge.srcAttr._2 > tol) {
        Iterator((edge.dstId, edge.srcAttr._2 * edge.attr))
      } else {
        Iterator.empty
      }
    }

    def messageCombiner(a: Double, b: Double): Double = a + b

    // The initial message received by all vertices in PageRank
    val initialMessage = resetProb / (1.0 - resetProb)

    // Execute a dynamic version of Pregel.
    Pregel(pagerankGraph, initialMessage, activeDirection = EdgeDirection.Out)(
      vertexProgram, sendMessage, messageCombiner)
      .mapVertices((vid, attr) => attr._1)
  } // end of deltaPageRank
}
```


Reference
---------

1. [PageRank-Wikipedia](http://en.wikipedia.org/wiki/PageRank)
2. [GraphX Official Guide](http://spark.apache.org/docs/latest/graphx-programming-guide.html#pagerank)
