## Solving compass angle in Simba 2.0

A fun walkthrough for solving for a relative angle between two vectors using the game Runescape as our  test-suit.

Let's just jump straight into the solving the problem. Note that a more accurate but complicated version exists here: https://pastebin.com/Dr42Cq1w

We can use the compass' south dot as our vector `q` to compare against vector `p` which is the middle of the compass (the point the compass rotates around) in order to find `ø`.

![p and q vectors](https://slacky.one/images/compass.png)

![It's tiny](https://slacky.one/images/tiny-compass.png)



As the compass is quite small our solve wont be very accurate, the radius from center to any dial is about 15px, theoretically this gives a radius of `2*π*15` or `94` distinct points that the southern dial will rotate through. In terms of accuracy this gives us `360/94` degrees error, or about `±3.8` degrees off. 

Now let's look at the code, and the math needed. So angles can be solved fairly simply in programming using **ArcTan2** function often available in most programming languages, sometimes called **atan2**.

> In trigonometry, **arctan** is the inverse tangent function. Inverse
> trigonometric can be recognized easily by their prefix **arc**, in
> math you will often see inverse tan described as **tan⁻¹**.

Initially we could try to solve this using just arctan like so:
```pascal
t := ArcTan((q.y-p.y) / (q.x-p.x))
```
However this will be problematic whenever X in `p` and `q` vectors are equal, as this will cause Division by Zero (`q.x-p.x=0`). This is where **ArcTan2** comes in, all those special cases will be handled automatically for us by doing a lot of if-tests.

```pascal
t := ArcTan2(q.y-p.y, q.x-p.x)
```
**t** which is short for **theta**, the Greek symbol **θ** used in mathematics to define angles is now returned in radians, as angles are usually measured in radians, but as we mainly want to work we degrees we have to follow up with converting to degrees, luckily Simba has a function called `RadToDeg` for us to use.

----

### Moving on to solving the problem in Runescape
first we have to define a function, let's call it **CompassAngle**
```pascal
function CompassAngle(): Single;
```

I am going to work in resizable client, so compass can be found at an offset from the right, this is fairly simple to deal with in Simba 2.0. And we have to define the bounds where we search for the colors of the dials, and also the north **N** which will be used to locate the southern dial.

```pascal
var
  middle: TPoint := [Target.Width-160,22];
  bounds: TBox   := Box(middle, 14, 14); //defines a box 29x29 around middle
```

We also know we need a TPoint to hold the northern coordinate, a TPoint to hold the southern coordinate once found, and TPointArray to hold the dials, and the ~3 pixels that's the south dial.
```pascal
  north, south: TPoint;
  N, dials, southtpa: TPointArray;  
```

&nbsp;
&nbsp;

The idea is minimalistic, ignoring basic error handling, we create pure functional code to keep this easier to follow. Let's do this in a step by step:


0) Find all matches of the color of the dials representing **west**, **south**, and **east**.
```pascal
  // 920735 is the integer of the red shade dials
  dials := Target.FindColor(920735, 1, bounds); 
```

1) As there are three dials all equally red, **west**, **south**, and **east**, and we only want the **southern** one we have to do some filtering of the data **dials**. So we find the northern **N** as this is unique.
```pascal
// 1911089 is the integer of the brown shade of N
N     := Target.FindColor(1911089, 1, bounds); 
```

2) Finding the **N** may cause some of what's outside the compass itself to match the color which is otherwise unique in the compass, and we only want that **N**, so we have to do a little radius filtering which can be achieved using [TPointArray.ExtractDist](https://villavu.github.io/Simba/api/TPointArray.html#tpointarray-extractdist). Now we generate a single point from the center of the **N** which we can work with, we call that **north**.
```pascal
// remove everything further away then 20px from center
// what remains is only the N, grab the mean of it.
north := N.ExtractDist(middle, 0,20).Mean();
```   

3) Using **north**-point we can remove all points that are not far enough away from it, thereby filtering out **west** and **east** points which are both closer to the **N** than the south dial. Again we will be using [TPointArray.ExtractDist](https://villavu.github.io/Simba/api/TPointArray.html#tpointarray-extractdist) to achieve this. 
```pascal
// the points that are further away than 20px from the N, this is South
southtpa := dials.ExtractDist(north, 20,9999);
```

4) We now have the **south** dial, but it's still an array of up to 3 pixels, stored in the **southtpa** variable - Next we want the single furthest pixel in **southtpa** from **middle**. This is where we can use a function in Simba called [TPointArray.FurthestPoint](https://villavu.github.io/Simba/api/TPointArray.html#tpointarray-furthestpoint). This function returns the point furthest away from.
```pascal
// extract the two furthest point in the south group
south    := southtpa.FurthestPoint(middle);
```

----

#### finally we can compute the angle
Only now can we compute the angle between the vectors **south** and **middle**, but using some offsets so that we compute the angle from the top, rotating clockwise. This as talked about earlier is a matter of using ArcTan2 function `ArcTan2(south.y-middle.y, south.x-middle.x)`, however we want to measure the angle from **north**, not from **west**, so we have to offset the angle by PI/2, or 90 degrees.

```pascal
ArcTan2(south.y-middle.y, south.x-middle.x) + PI/2
```
Still this is not enough, what I have forgotten to mention is that ArcTan2 returns the angle as radians, between `-π..+π`, or `-180..180` converted to degrees. We can use the Modulo function in Simba 2.0 (not to be confused with the **mod** operator in Pascal) to wrap it into a `0..360` range.

```pascal
degrees := RadToDeg(ArcTan2(south.y-middle.y, south.x-middle.x) + PI/2)
Result := Modulo(degrees, 360);
```


------
------


## Final code
```pascal
function CompassAngle(): Single;
var
  middle: TPoint := [Target.Width-160,22];
  bounds: TBox   := Box(middle, 14, 14);
  north, south: TPoint;
  N, dials, southtpa: TPointArray;
begin
  dials := Target.FindColor(920735,  1, bounds);
  N     := Target.FindColor(1911089, 1, bounds);
  north := N.ExtractDist(middle, 0,20).Mean();

  southtpa := dials.ExtractDist(north, 20,9999);
  south    := southtpa.FurthestPoint(middle);

  Result := Modulo(RadToDeg(ArcTan2(south.y-middle.y, south.x-middle.x)-PI/2), 360);
end; 
```
