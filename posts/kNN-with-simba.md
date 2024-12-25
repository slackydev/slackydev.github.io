#### Simba 2.0: Entering the Realm of Machine Learning
Simba 2.0 has traditionally relied on classical methods for object detection. However, with the new addition of a powerful KDTree data structure and the KNearestClassify-method, Simba 2.0 now enters the realm of Machine Learning. This development empowers users to leverage the power of k-Nearest Neighbors (kNN) for a range of image analysis tasks. 

#### The Versatility of kNN 
The k-Nearest Neighbors (kNN) algorithm, a core concept in Machine Learning, has a long history starting in the 1950s. It operates by classifying new data points based on the labels of their closest neighbors in a dataset. 
This versatile technique finds applications across various domains. From powering recommendation systems on platforms like Netflix and Amazon to assisting in medical image analysis and even driving financial forecasting, kNN demonstrates the power of Machine Learning to uncover hidden patterns and make intelligent predictions. 

&nbsp;

## Exploring kNN
In this article, we delve into the practical application of kNN within the Simba 2.0 environment by exploring its implementation in classifying handwritten digits from the renowned MNIST dataset. I'll guide you through the algorithm's inner workings, analyze its performance, and discuss the challenges and considerations. This exploration will provide valuable insights into the power and versatility of this machine learning algorithm.


**kNN classification**

kNN classification finds the 'k' nearest neighbors to a new data point within the dataset. Imagine a set of points representing different fruits, like apples and cucumbers. Each point has two properties:

-   **Height:** The length of the fruit (up and down on the graph).
-   **Width:** The thickness of the fruit (left to right on the graph).

Each point is labeled with its fruit type (e.g., "apple" or "cucumber"). To classify a new fruit, kNN finds the 'k' closest fruits in terms of height and width. The new fruit is then assigned the label that appears most often among its 'k' nearest neighbors.

For example, if the three closest fruits to a new fruit are two apples and one cucumber, the new fruit would be classified as an apple.

**Visualizing with a Graph:**

Red points represent apples, and green points represent cucumbers. In this graph, apples tend to be clustered around the same height and width (around 9cm for both). Cucumbers, however, are longer and thinner, spread out in a line between 12cm and 22cm in height but only 3cm to 7cm wide. The image also includes both standing and laying cucumbers to account for orientation variations.

![Apples and cucumbers](https://github.com/user-attachments/assets/b5e39ab7-a450-465e-a8c3-45fb4116eb6f)


**Querying the Classifier:**

-   If we query a point like [15, 6] (high height, low width), the kNN classifier will find mostly cucumbers as its nearest neighbors due to their similar heights.
-   Even if we reverse the order to [6, 15], the closest neighbors might still be cucumbers because kNN considers both height and width, accounting for the rotation thanks to our training data including both orientations.
-   On the other hand, a query point like [7, 7] (similar height and width) would likely have apples as its nearest neighbors, classifying the new fruit as an apple.

**Limitations:**

-   kNN doesn't predict "unknown." If a query point like [0, 0] is far from all data points, kNN will still classify it based on the nearest neighbors, even if they aren't very close.

**Note**

- This example uses only two dimensions (height and width) for simplicity. However, kNN can be applied to datasets with many more dimensions. For example, to classify images, you could use features like color (represented by red, green, and blue values). This would result in data points with higher dimensionality, such as [width, height, R, G, B]. By considering multiple features, kNN can make more accurate and nuanced classifications.

&nbsp;

#  kNN with Simba 2.0
Now that you hopefully understands the base principles of kNN we can look at how we solve something like [MNIST](https://yann.lecun.com/exdb/mnist/)  dataset.

To start we need to be able to parse the [MNIST](https://yann.lecun.com/exdb/mnist/) dataset, for this I have written a couple functions that can be viewed here: https://pastebin.com/H3eEMa0G

While the dataset itself can be found many places, here is one set (top google result): https://www.kaggle.com/datasets/hojjatk/mnist-dataset

```pascal
var
  Images,TrainImages: array of TSingleMatrix;
  Labels,TrainLabels: array of Byte;
begin
  WriteLn('>>> Parsing MNIST dataset');
  TrainImages := ParseMNISTFile('Scripts\MNIST\train-images.idx3-ubyte');
  TrainLabels := ParseMNISTLabels('Scripts\MNIST\train-labels.idx1-ubyte');

  Images := ParseMNISTFile('Scripts\MNIST\t10k-images.idx3-ubyte');
  Labels := ParseMNISTLabels('Scripts\MNIST\t10k-labels.idx1-ubyte');
end; 
```

All the letters in the images are grayscale, 0..255, and 24x24 in size, we store it in a TSingleMatrix for rows[24] and columns[24], rather than a TImage. 

We get two sets, one is 50K images and Labels (0..9) and the second one is for us to test on with 10K images and labels that have not been trained on.

My approach to extracting features is very simplistic, I divide the image into 4x4 quadrants, then I count the number of pixels in each quadrant. This gives us 16 features to use, or 16 dimensions. 

```pascal
(*
  Parse a single image into a 16 features, which is just us counting
  the number of pixels in a quadrant of the shape.
  
  As the image is antialiased (smooth) we threshold by simply
  extracting pixels that have more than 64 brightness.
  
  Each feature is normalized to the range 0..1
*) 
function VectorDataBlock(Img: TSingleMatrix): TSingleArray;
var
  TPA: TPointArray;
  tmp: T2DPointArray;
  Bounds: TBoxArray;
  i: Int32;
begin
  SetLength(Result, 4*4);
  TPA := Img.Indices(64, __GT__);
  Bounds  := TPA.Bounds.Partition(4,4);
  for i:=0 to High(Bounds) do
    Result[i] := TPA.ExtractBox(Bounds[i]).Length / ((24*24) / (4*4));
end;

(* 
  Do the above for every image in our set
*)
function ComputeVectors(Images: array of TSingleMatrix; Labels: TByteArray): TKDItems;
var i: Int32;
begin
  SetLength(Result, Length(Images));
  for i:=0 to High(Images) do
  begin
    Result[i].Ref := Labels[i];
    Result[i].Vector   := VectorDataBlock(Images[i]);
  end;
end; 
```

Now we can compute the feature vectors for every number in our test and training data

```pascal
var
  TrainingData, TestData: TKDItems;
begin
  WriteLn('>>> Generating features');
  TrainingData := ComputeVectors(TrainImages, TrainLabels);
  TestData     := ComputeVectors(Images, Labels);
end;  
```

Finally we can now generate a KDTree for our classification. KDTrees are well suited for kNN as they provide a very fast way to get the `k` nearest neighbors of some input, but it does take some time to build.

```pascal
var
  Tree: TKDTree;
begin
  WriteLn('>>> Training the lookup tree with features');
  Tree.Init(TrainingData);
end; 
```

Finally we can use it by calling **Tree.KNearestClassify**:

```pascal
var
  i, category, hits: Int32;
begin
  WriteLn('>>> Testing against test-set');
  for i:=0 to High(TestData) do
  begin
    if i mod 1000 = 999 then
      Writeln(i+1,' -> Accuracy: ', Format('%.3f', [hits / (i+1)*100]),'%');

    category := Tree.KNearestClassify(TestData[i].Vector, 7);
    if category = TestData[i].Ref then
      Inc(hits);
  end;

  WriteLn('MNIST Accuracy: ', hits / Length(TestData)*100,'%');
end; 
```

----

## Impressive accuracy 
The **kNN model** we just made achieved an impressive **94%** accuracy on the MNIST dataset, demonstrating its effectiveness for handwritten digit recognition. This result is comparable many simpler neural networks while being much easier to implement.

&nbsp;
/slacky out
