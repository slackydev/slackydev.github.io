## Solving doors in RS by Anchor

Code: https://gist.github.com/slackydev/9da5b818db0ecd89aa846af999db5314

You can see it in action here:
https://i.giphy.com/8nwVZiukGLZe5njsmE.webp

This solution assumes that you have an anchor that is either perpendicular or parallel to the door. 

**Closed door as parallel to an anchor**

![image](https://github.com/user-attachments/assets/3b05dc9f-d68c-4d5d-8e97-bcf658ad8597)

**Open door as perpendicular to anchor showing it's distance**

![image](https://github.com/user-attachments/assets/bad41b92-d08d-4ee8-b7c3-49bfc7e7a92d)

This is the basis of this concept.

The function itself looks like:
```pascal
function IsDoorOpen(Door, Anchor: TPointArray; Split: Double; Inverse: Boolean = False; UseMean: Boolean = False): EDoorState;
```

Argument description:
```
  Input: 
  Door:   The TPA of the door
  Anchor: The TPA of the Anchor
  Split:  This is the cutoff-point between a closed and an open door
  Inverse: If the door is not parallel then this should be True (auto False)
  UseMean: Mean may give less accurate distance value, so it may result in (more) false-positives
           But it may be less prone to noisy pixels from color-finding.  
           
  Return `dsOpen` if the door is open based on given input data 
  if the method fails it will return `dsUndefined`, otherwise dsFalse; `ds` is short for doorstate 
```

-----

So for our example we get very a clear cutoff-point. Open door is registered as 75px, and a closed door registers as 90px. 
The difference between the two states is `90-75=15px`. You may therefor now think that `7px` or `8px` is the cutoff, however it's 
a bit more involved than that. The code now transforms these two coordinates into 2d space using minimap-to-mainscreen. This transform makes the code invariable to zoom level, since the minimap isn't affected by zoom.

So at the top of the code you find
> `{$DEFINE DOOR_DEBUG_DISTANCE}`

This plots the actual distance number in 2d minimap space from anchor to the door. So in this example we get:

> `10.58017063` and `14.25140381` pixels in 2d minimap space.

Now the cutoff is roughly the middle of those usually. 
```
> 14.25140381 - 10.58017063 = 3.67123318
> 3.67123318 / 2            = 1.83561659
> 10.58017063 + 1.83561659  = 12.41578722
``` 
We now know the cutoff is about 12.4

> `state := IsDoorOpen(Door, Anchor, 12.4, False, False);`

When we set `UseMean=False` the method will use the closest points in the two TPAs. So the pixel in the anchor that is nearest the door, and the pixel in the door that is nearest the anchor. This produces a very reliable, consistent distance measure when you can reliably capture the door and anchor TPAs.

![image](https://github.com/user-attachments/assets/3b05dc9f-d68c-4d5d-8e97-bcf658ad8597)
![image](https://github.com/user-attachments/assets/bad41b92-d08d-4ee8-b7c3-49bfc7e7a92d)

-----

If we set `UseMean=True` then the middle point of the door and the middle point of the anchor will be used for our distance measurement, as shown in the images.

![image](https://github.com/user-attachments/assets/1b803ee0-ae34-4b0e-a801-f2d61f277e4a)
![image](https://github.com/user-attachments/assets/aa95dee1-9f46-46c0-af21-aa71c97eaffb)

Also here we got very good difference between the anchor and door in the states, 108px, and 93px. But this is actually more prone to error as the TPA will fluctuate more with varying distance to the door and different compass angles, since the mean is computed in two dimensions, while the game distance and angle is actually projected in 3 dimensions. So you have been warned. We counter some of this error by converting to minimap space, but this works much better for when we dont set `UseMean=True`.

```
> 16.11234665 - 14.2799902 = 3.67123318
> 1.83235645 / 2           = 0.916178225
> 14.2799902 + 0.916178225 = 15.196168425
``` 

> `state := IsDoorOpen(Door, Anchor, 15.2, False, True);`

-----
Let's have a peek at the finder code, you can use your own finders, but i'll just walk through the finding that is built in.

```pascal
procedure FindDoorAndAnchor(WorldPos: TPoint; out Door, Anchor: TPointArray);
var
  ledger: TPointArray;
  ATPA: T2DPointArray;
  AnchorSearchBox, DoorSearchBox: TBox;
begin
  // rough area of where the anchor and door is (world map coordinates)
  AnchorSearchBox := RSW.GetTileMSEx(WorldPos, [4262, 1964]).Expand(30).Bounds();
  DoorSearchBox   := RSW.GetTileMSEx(WorldPos, [4248, 1958]).Expand(30).Bounds();

  DoorSearchBox.LimitTo(MainScreen.Bounds);
  AnchorSearchBox.LimitTo(MainScreen.Bounds);

  // find the anchor, and the door tpas (regular color-data from ACA)
  srl.FindColors(anchor, CTS2(12308179, 1, 0.01, 0.01), AnchorSearchBox);
  srl.FindColors(door,   CTS2(2060172, 10, 0.06, 1.33), DoorSearchBox);

  // basic noise filtering
  ATPA := door.Erode(1).Grow(1).Cluster(2);
  door := ATPA.Biggest();

  // basic noise filtering, small object, so I dont erode noise.
  ATPA := anchor.Cluster(2);
  anchor := ATPA.Biggest();
end;
```

First we define where the anchor is in terms of world coord, and where the door is in terms of world coord.
[4262, 1964], [4248, 1958] are the world coord positions for anchor and door respectively. Replace this with your locations.
```pascal
  AnchorSearchBox := RSW.GetTileMSEx(WorldPos, [4262, 1964]).Expand(30).Bounds();
  DoorSearchBox   := RSW.GetTileMSEx(WorldPos, [4248, 1958]).Expand(30).Bounds();
```

```pascal
function TRSWalker.GetTileMSEx(Me, Loc: TPoint; Height:Double=0; Offx,Offy:Double=0): TRectangle; 
```
GetTileMSEx takes a high as well, for rough estimate, if your anchor is higher up than ground level you probably want to pass it's height (roughly). The top of a desk is about `4` high, while the top of a door is about `8` high. Often you can just drop passing height for our usage, but tweaking these may actually result in slightly better finding, notably when you pass anchor's height. Door is not as important, but passing 4 (average of the door height) may yield better finding. 

But instead if that I actually just call .Expand(30) for both in the above code to get a larger search area. You can therefor leave this as is I guess.

```pascal
  // find the anchor, and the door tpas (regular color-data from ACA)
  srl.FindColors(anchor, CTS2(1776416, 1,  0.01, 0.01), AnchorSearchBox);
  srl.FindColors(door,   CTS2(1993096, 13, 0.08, 1.02), DoorSearchBox);
```
Then there is the color-finding, as described you want to pass it the color of the anchor (first one) and the color of the door (second one). For  this ACA in simba will help you a lot! You want to do this for both the anchor and the door.

![image](https://github.com/user-attachments/assets/c773c1cf-b322-4bd7-a5a7-aabf68a525c9)

----

Picking an anchor, anchors should be an easy to find item that is in visible range from where you want to open the door from, that is also visible from the angles your script operates from. Often you want to pick something very distinctive, i chose the sign in my example since that sign will _never_ change color, it will always be in the same position as well, and is perfectly parallel to the closed door, this is an optimal result.

Anchor should always be parallel to the closed door, or parallel to the door when it's open. Meaning the knob on the door should point towards the anchor. Naturally finding such a perfect anchor is not always possible. But if the anchor is facing away from the door(knob) when it's closed it will not work at all.

The regions to look for an anchor can be look at like this:

![image](https://github.com/user-attachments/assets/938747dc-8ba0-42a4-b47c-6b9b0ce75b58)

The curve marked in black is the direction of the door, how it opens in my example. Due to this the arc marked in green is where you want to look for an anchor. While the area marked in red will cause the anchor to fail. 
This is not a perfectly accurate description, but should give you a decent pinpoint when solving for your door. Generally speaking, just avoid getting close to diagonals, as they will not work with this design, a diagonal anchor will cause the distance to never change between door states.

----

When dealing with anchors that get closer to a diagonal, because you can't find any better reliable anchor you may need to rotate the camera a few times while debugging the distances, and tweak the split distance. This can often be done by eyeballing it, at least after a bit of practice.

---

Looking back at the function definition:
```pascal
function IsDoorOpen(Door, Anchor: TPointArray; Split: Double; Inverse: Boolean = False; UseMean: Boolean = False): EDoorState;
```

I want to look at the parameter named `Inverse`, the code assumes that the closed door is parallel to an anchor (knob points towards anchor). However if the door is open when it is parallel to the anchor you need to set `Inverse` to True. 
