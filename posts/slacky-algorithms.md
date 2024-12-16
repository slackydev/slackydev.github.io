I thought I'd share some work, as described I dabble a fair bit in [Computational geometry](https://en.wikipedia.org/wiki/Computational_geometry) and I have implemented numerous custom algorithms for well known problems, often problems that are not as well defined, but also those which are.

Here's a snapshot of some algorithms I've made over the years.

[Skeletonization](https://en.wikipedia.org/wiki/Topological_skeleton)
-----------------------------------------------------------------------
Efficient algorithm for skeletonization - does not guarantee connected skeleton but usually produces one.

![image](https://github.com/user-attachments/assets/e6f0dcd4-ab33-49b4-8662-adca72f38e03)

https://gist.github.com/slackydev/316cc8f862b33108222aab8f1cc0da72
https://github.com/Villavu/Simba/blob/3276def9ba9c306360001400d796c6c2d7998ea9/Source/simba.vartype_pointarray.pas#L2456



[Clustering](https://en.wikipedia.org/wiki/Cluster_analysis)
------------------------------------------------------------
Efficient spatial 2d clustering algorithm for separating groups of points by distance.

![image](https://github.com/user-attachments/assets/d336bae1-1dfc-43f1-8070-540a480cb046)

https://github.com/slackydev/SimbaExt/blob/master/DLL%20Source/SimbaExt/src/PointTools.pas#L1493-L1501

This algorithm has been used for various computer vision tasks, and performs well, and has been optimized more by others in separate implementations.

Just for fun I've made a very simple recursive O(n^2) variant:
https://gist.github.com/slackydev/fcfd4bf27f3eb3c90c465cc8fe6e8952

Which can actually be implemented using a KD-tree (here in Pascal), even though it uses a KD-tree, this variant also grows roughly quadratic in worst case which happens when most or all points are a single group:
https://gist.github.com/slackydev/1e4c6131ae78f7a50cab9ecd745c2030

[MinAreaCircle](https://en.wikipedia.org/wiki/Smallest-circle_problem)
----------------------------------------------------------------------
Efficiently solving the problem of generating a minimum circumference circle covering any set of points.

![image](https://github.com/user-attachments/assets/d8e7a103-fd1a-4d5d-9687-db3f8c677734)

https://github.com/Villavu/Simba/blob/3276def9ba9c306360001400d796c6c2d7998ea9/Source/simba.vartype_pointarray.pas#L2573-L2594


[ConcaveHull](https://en.wikipedia.org/wiki/Alpha_shape)
--------------------------------------------------------
Two approximating algorithms for generating a fitting concave polygon from a set of points. Both images are generated at default parameters.

![image](https://github.com/user-attachments/assets/2c28923e-0d4d-4ef8-a47c-ec0eeaf62de8)

https://gist.github.com/slackydev/5e6ac4b07cbec895baf4e7eb0faeff13


ConvexityDefects
------------------
Image shows convexity defects based on concavehull algorithm shown above, wrapped with convexhull.

![image](https://github.com/user-attachments/assets/fc93a965-5e51-41a1-9275-40d14c0c65c4)


https://gist.github.com/slackydev/fe5626292a8d0fedd88444f578d12803
