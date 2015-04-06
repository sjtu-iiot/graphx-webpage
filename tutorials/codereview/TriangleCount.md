Triangle Count in GraphX
========================
Triangle Count is very useful in social network analysis. For example, in twitter or weibo, if the person you followed also follows you, there can be a lot of triangles in the social network. It means that your social network has tight connections and the community is strong enough.

A vertex is part of a triangle when it has two adjacent vertices with an edge between them. GraphX implements a triangle counting algorithm in the `TriangleCount object` that determines the number of triangles passing through each vertex, providing a measure of clustering.

Algorithm
----------
`TriangleCount` is defined in [TriangleCount.scala]. It counts the triangles passing through each vertex using a straightforward algorithm:

1. Compute the set of neighbors for each vertex;

2. For each edge compute the intersection of the sets and send the count to both vertices;

3. Compute the sum at each vertex and divide by two since each triangle is counted twice.

Suppose `A` and `B` are neighbors. The set of neighbors of `A` is `[B, C, D, E]`; the set of neighbors of `B` is `[A, C, E, F, G]`. And the intersection is `[C, E]`. The vertices in the intersection are their common neighbors, so `[A, B, C]` and `[A, B, E]` are two triangles.

[TriangleCount.scala]:https://github.com/apache/spark/blob/master/graphx/src/main/scala/org/apache/spark/graphx/lib/TriangleCount.scala#L1

Using TriangleCount API
---------------

`TriangleCount` requires the edges to be in canonical orientation (srcId < dstId). Also the graph must have been partitioned by using [Graph.partitionBy].

Therefore, we need to specify the `canonicalOrientation` as `true` when importing the graph, and partition the graph with `partitionBy()`. Use the API as the following:

### Import the libraries:

  ```scala
  scala> import org.apache.spark._
  scala> import org.apache.spark.graphx._
  scala> import org.apache.spark.rdd.RDD
  ```
### Load the edges in canonical order and partition the graph for triangle count  :

  ```scala
  scala> val graph = GraphLoader.edgeListFile(sc,hdfs://192.168.17.240:9222/input/yuhc/web-Google/web-google.txt",true).partitionBy(PartitionStrategy.RandomVertexCut) 
  ```
### Find the triangle count for each vertex :

  ```scala
  scala> val triCounts = graph.triangleCount().vertices 
  ```
### Show the result :

  ```scala
  scala> triCounts.take(10) 
  res0: Array[(org.apache.spark.graphx.VertexId, Int)] = Array((354796,1), (672890,0), (129434,1), (194402,2), (199516,163), (332918,6), (170792,24), (386896,129), (691634,566), (513652,1))
  ```

[Graph.partitionBy]:http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.graphx.Graph@partitionBy(PartitionStrategy):Graph[VD,ED]

Source Code Analysis
---------------

The codes of `TriangleCount` is shown as follows. Some comments are added to help the understanding.

  ```scala
  object TriangleCount {
 
  def run[VD: ClassTag, ED: ClassTag](graph: Graph[VD,ED]): Graph[Int, ED] = {
    // Remove redundant edges
    // web-Google has no redundant edges
    val g = graph.groupEdges((a, b) => a).cache()
 
    // Construct set representations of the neighborhoods
    val nbrSets: VertexRDD[VertexSet] =
      g.collectNeighborIds(EdgeDirection.Either).mapValues { (vid, nbrs) =>
        // Why the code specifies the capacity of `set`?
        val set = new VertexSet(4)
        var i = 0
        // Store the neighbors in the VertexSet
        while (i < nbrs.size) {
          // prevent self cycle
          if(nbrs(i) != vid) {
            set.add(nbrs(i))
          }
          i += 1
        }
        set
      }
    // join the sets with the graph
    val setGraph: Graph[VertexSet, ED] = g.outerJoinVertices(nbrSets) {
      (vid, _, optSet) => optSet.getOrElse(null)
    }
    // Edge function computes intersection of smaller vertex with larger vertex
    def edgeFunc(ctx: EdgeContext[VertexSet, ED, Int]) {
      assert(ctx.srcAttr != null)
      assert(ctx.dstAttr != null)
      // Check whether the items in the `smallSet` are in the `largeSet`
      val (smallSet, largeSet) = if (ctx.srcAttr.size < ctx.dstAttr.size) {
        (ctx.srcAttr, ctx.dstAttr)
      } else {
        (ctx.dstAttr, ctx.srcAttr)
      }
      val iter = smallSet.iterator
      var counter: Int = 0
      // Enumerate the items
      while (iter.hasNext) {
        val vid = iter.next()
        if (vid != ctx.srcId && vid != ctx.dstId && largeSet.contains(vid)) {
          counter += 1
        }
      }
      ctx.sendToSrc(counter)
      ctx.sendToDst(counter)
    }
    // compute the intersection along edges
    val counters: VertexRDD[Int] = setGraph.aggregateMessages(edgeFunc, _ + _)
    // Merge counters with the graph and divide by two since each triangle is counted twice
    g.outerJoinVertices(counters) {
      (vid, _, optCounter: Option[Int]) =>
        val dblCount = optCounter.getOrElse(0)
        // double count should be even (divisible by two)
        assert((dblCount & 1) == 0)
        dblCount / 2
    }
  } // end of TriangleCount
}
  ```
