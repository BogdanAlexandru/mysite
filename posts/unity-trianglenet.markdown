---
title: Random Locations within Polygons in Unity
date: 2018-01-20
---

This post follows a discussion on **/r/gamedev's** Discord server, which was about giving NPCs large areas to roam within, as a possible alternative to using separate agent types & walkable areas (part of Unity's builtin NavMesh tools).

This is a fairly rough prototype that can be further improved in a real project, especially w.r.t. the interaction with Triangle.Net and the editor experience. The code I used is [here](https://github.com/BogdanAlexandru/UnityTriangleNetExample), feel free to improve on it but since it's really just about three non-glue scripts with very few functions, I recommend writing your own as you learn the ideas from the article.

Scenario: you want to define a polygonal area in your scene and pick random locations on it, for example to have a NPC roam within. There's no one-shot solution for this that I'm aware of, but there's at least one method for picking uniform random points within a triangle. Since polygons can be triangulated, we can apply the triangle formula in the context of the polygon.

### Note:
This is not a fast operation, if your plans involve triangulating areas every frame you may want to find something else or figure out a way to either speed it up or do it in the background so it doesn't kill the game's framerate.

There may also be much simpler alternatives that get you enough of the way there:

* You can specify a roaming area as a sphere. `UnityEngine.Random.insideUnitSphere * yourRadius` gives you a random point inside a sphere with `yourRadius` radius. Snap the point to your terrain.
* Your roaming area is a rectangle. You have `Width * Height` possible locations if you use a distance of `1` between locations. Picking a random location does not involve anything more than a few mathematical operations. Think 2D arrays represented as a list.
* You can generate equidistant points within your polygon. This requires extra memory unless you can find a way to perform the generation on an as-needed (i.e. lazy) basis, similar to what I mentioned for the rectangle above.
* You can specify the locations manually — waypoints. A few waypoints and a NPC that cleverly picks between them will likely result in a pleasant behavior, with only a fraction of the effort. Make him interact in a nice way with other entities on his route and you've got yourself an amazing NPC.
* The player likely won't notice that your characters don't ever go at `(2.5, 3)` and instead always sit in `(2, 3)` and `(3, 3)`. Meaning, you may really only care about a small subset of the possible locations in the polygon.

Finally, keep in mind that as long as your areas don't change at runtime, you can triangulate/generate points in the editor and assign the results to fields on your scripts. This moves all polygon processing overhead to the editor, which is nice. A baking of sorts.

### The Solution

Given a triangle with vertices `A`, `B` and `C`, you can find a random point `P` within the triangle with the formula:

\\[ P = (1 - \\sqrt{r_1}) A + (\\sqrt{r_1} (1 - r_2))  B + (r_2 \\sqrt{r_1}) C  \\]

where `r1` and `r2` are uniformly distributed random numbers in `[0, 1]` i.e. as returned by `UnityEngine.Random.Range(0f, 1f)`.

<center>
![](/images/triunif.png)
*GNU Octave: 1000 points within `[(0, 0), (0, 1), (1, 0)]`*
</center>

We can apply the formula in the context of any polygonal shape if said shape is triangulated. For the triangulation process you can use an existing library that already implements [Delaunay triangulations](https://en.wikipedia.org/wiki/Delaunay_triangulation), such as [Triangle.net](https://archive.codeplex.com/?p=triangle), or write your own (it's not trivial, in case you're not sure you've got the time to spend). 

I gave Triangle.net a shot and it appeared to work just fine for what I threw at it. The official distribution should work fine now that Unity supports .NET 4.6 but to avoid any kind of hassles I used [this fork](https://github.com/BogdanAlexandru/Triangle.NET) which was already confirmed to work with Unity.

Before we start throwing triangles around though, we need a polygon.

## Setting Up the Polygon

An `Area` script can be defined, which will have a `List<Vector3>` of vertices that describe the polygonal shape. Each vertex is connected to the one after it, and the last one connects to the first (so the order in which they're added to the list matters).

```cs
public class Area : MonoBehaviour
{
    public List<Vector3> Corners;
    //                   or vertices

    // extra stuff...
}
```

 We want to be able to edit this polygon in the scene view, so a custom editor for the `Area` component needs to be written.

[This tutorial](http://catlikecoding.com/unity/tutorials/curves-and-splines/) that deals with Bezier curves, or at least the first quarter of it, teaches all that you need to get this done, so if you're not sure how to proceed, give it a shot. In short, we're not doing much more than drawing lines between points and using handles to move said points around.

This is the end result for me (note that here the polygon is already being triangulated):

<center><a target="_blank" href="/images/areaa.png"><img style="width:600px" src="/images/areaa.png" /><a/></center>

Or in an animated form:

<center><a target="_blank" href="/images/trianggif.gif"><img src="/images/trianggif.gif" /><a/></center>

Notice that we're ignoring the `Y` coordinate. We're triangulating a 2D polygon and in the current orientation we only care about the `X` and `Z` coordinates. See the questions section at the end of the article w.r.t. dealing with terrain height. For a more robust editor experience, you'll want to adapt the polygon preview to the terrain's height as well.

In the end, you want the `Area` script to expose a `PickRandomLocation` function or similar, allowing calls like:

```cs
class RoamingAI : MonoBehaviour {

    // Map this to the area you need in the inspector
    public Area RoamingArea;

    // later on...
    var location = RoamingArea.PickRandomLocation();

    // ...
```

## Triangulating the Polygon

With a set of vertices that describe the polygon, we can ask Triangle.Net to do some triangulation for us.

This is my function that calls into Triangle.Net:

```cs
public static ICollection<Triangle> Triangulate(
    IEnumerable<Vector2> points)
{
    var poly = new Polygon();
    poly.Add(new Contour(points.Select(p => p.ToVertex())));
    var mesh = poly.Triangulate();
    return mesh.Triangles;
}
```

The `Vector2`s given as input are the `Vector3` vertices of the `Area`, with the `z` coordinate taking the place of `y`. In other words, `area.Vertices.Select(v => new Vector2(v.x, v.z))`.

`Polygon` is a Triangle.Net type, as are `Contour` and `Vertex`. We turn a `Vector2` into a `Vertex` via an extension method called `ToVertex` that does nothing more than `new Vertex(vec2.x, vec2.y)`. Then, we build a `Contour` out of our vertices and add it to the `Polygon`.

With the `Polygon` set up, all that's left is to call `Triangulate()` on it, which gives us back an `IMesh` object, from which we only need the collection of `Triangle`.

## Picking a Random Triangle

With the collection of triangles calculated, we can pick one to apply the random point formula on. We want the random positions to be picked as uniformly as possible, so simply choosing one of the triangles at random won't do:

<center><a target="_blank" href="/images/wounif.png"><img style="width:600px" src="/images/wounif.png" /><a/></center>
<center>*1000 positions: `Random.Range(0, _triangles.Count)` is not ideal*</center>

Instead we'll introduce some bias, so larger (by area) triangles become proportionately more likely to be selected.

```cs
// The sum of the areas of the triangles can be cached.
_areaSum = 0f;
_triangles.ForEach(t => _areaSum += t.TriArea());

// ...

private Triangle PickRandomTriangle()
{
    var rng = Random.Range(0f, _areaSum);
    for (int i = 0; i < _triangles.Count; ++i)
    {
        // TriArea() is an extension method
        // that uses the cross product formula
        // to calculate the triangle's area.
        if (rng < _triangles[i].TriArea())
        {
            return _triangles[i];
        }
        rng -= _triangles[i].TriArea();
    }
    // Should normally not get here
    // so this is not 100% correct.
    // You'd need to consider floating
    // point arithmetic imprecision.
    //
    // But this is a good compromise 
    // that nonetheless gives good 
    // results.
    return _triangles.Last();
}
```

<center><a target="_blank" href="/images/wunif.png"><img style="width:600px" src="/images/wunif.png" /><a/></center>
<center>*With bias: much better!*</center>

## Picking a Random Location

With a `Triangle` having been chosen, all that's left is to apply the aforementioned formula:

```cs
private Vector2 RandomWithinTriangle(Triangle t)
{
    var r1 = Mathf.Sqrt(Random.Range(0f, 1f));
    var r2 = Random.Range(0f, 1f);
    var m1 = 1 - r1;
    var m2 = r1 * (1 - r2);
    var m3 = r2 * r1;

    var p1 = t.GetVertex(0).ToVector2();
    var p2 = t.GetVertex(1).ToVector2();
    var p3 = t.GetVertex(2).ToVector2();
    return (m1 * p1) + (m2 * p2) + (m3 * p3);
}
```

The result is our random position within the polygon describing the `Area`. Done.

## Possible questions:

<div class="in-article-question">
<h4>How do I use this in a 3D world? My terrain surface is not flat.</h4>
<p>Once you have the random 2D coordinates generated, `(x, z)`, you can perform a raycast from `(x, HighEnoughY, z)` downwards. Once you hit the terrain, get your final position by taking `hitPoint.y` as your random point's `y` coordinate.</p>

<p>Remember to set your terrain on its own layer if it's not on one already, and make your raycast only hit the terrain layer via the layer mask argument. This will improve performance and reduce the chance that your character receives the top of a giant orc's head as destination.</p>

<h4>How do I add holes within the polygon?</h4>
<p>Aside from the area vertices, you will need another set of vertices for each hole. Once you have that, instead of adding just the area vertices to the `Polygon` that's to be triangulated, also add a `Contour` for each hole:</p>

```cs
var p = new Polygon();
p.Add(new Contour(verticesOfArea));
holes.ForEach(h => p.Add(new Contour(h.Vertices)));
var mesh = p.Triangulate();
```

For the editor part, you may want to separate the `Area` component into an `AreaPolygon` and `Area` pair, with the former only keeping track of the vertices that make up the polygon, and the latter taking in those vertices, triangulating the polygon and returning a random position. Add empty GameObjects as children to the `Area` GameObjects, and give them `AreaPolygon` components. The topmost `Area` component will gather all the `AreaPolygon` data from its direct children and consider each one as a hole inside its own polygon.

<center><a target="_blank" href="/images/holeex.png"><img src="/images/holeex.png" /><a/></center>
</div>
