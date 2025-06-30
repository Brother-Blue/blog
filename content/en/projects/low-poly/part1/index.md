---
title: A Low Poly Image Conversion CLI Tool
slug: low-poly-cli
description: |
  How I built a CLI tool to create one of my favorite aesthetic styles. Going into the research needed to make it happen and my future plans for the project.
date: 2025-06-27T02:04:37.937Z
preview: ""
draft: false
tags:
- CLI
- Go
- Because I could
categories:
- Media
- Art
authors:
- chris
series:
- "Low Poly"
series_order: 1
---

## The Background

As an engineer who suddenly found himself with too much time on his hands, I decided it would be time to get back into some long-forgotten side projects. The first on my list was this low poly converter. There's something oddly appealing about low poly art, to me if you apply it to the right image it gives a result that makes me understand the appeal of modern art.

Low poly images (low polygon images) are geometric images, that are commonly composed of triangles. They have sharp vibrant edges and when done right maintains a large amount of detail in the original image. An appeal of this is how much detail we can actually make out with less information.

I didn't want to be able to only convert static images though. I wanted to make this into a tool that might have some more interesting use cases in the future. It works on `jpg`, `png`, as well as `gif` files; and eventually I plan on making it work on video formats as well.

## The Inspiration

Aside from my niche interest in triangles, I wanted to challenge myself to build a tool and automate releases. I've of course done this plenty of times in the past at my previous work, but there were usually services and pipelines in place that never really needed updates.

This project, as small as it was, gave my the opportunity to create something new and to improve my skills with pipeline automation.

## The Code
All of the code is available at my GitHub.
{{< github repo="brother-blue/low-poly-converter" showThumbnail=false >}}

There's a few key components when creating low poly images, the first of which is to decide on how much variety you want in the result. To achieve this I decided on an arbitrary density. I use `poly point` and `triangulation point` interchangeably, they both mean a point that will be used in Delaunay triangulation. I added a minimum number of points to ensure there is some poly-fication of the image.
```go
// One poly point per X pixels
const density = 500

// Minimum of 10 triangulation points
const minPoints = 10
```

The next step was to include the four corners of the image. Without them there would be slim triangular artifacts along the edges of the resulting image. You can see them in the resulting image of [Issue #1](https://github.com/Brother-Blue/low-poly-converter/issues/1) in the project.
```go
// Generate random points
// Add + 4 to account for the corners
points := make([][2]float64, polyPoints+4)
for i := 0; i < polyPoints; i++ {
  x := rand.Float64() * float64(w)
  y := rand.Float64() * float64(h)
  points = append(points, [2]float64{x, y})
}

// Add corners of the image
points[polyPoints] = [2]float64{0, 0}
points[polyPoints+1] = [2]float64{float64(w - 1), 0}
points[polyPoints+2] = [2]float64{0, float64(h - 1)}
points[polyPoints+3] = [2]float64{float64(w - 1), float64(h - 1)}
```

Now that the random points have been added to the image, the next step is for actually figuring out the triangles. To do this I used a technique called. It works by creating circumcircles around a set of points forming a triangle - ensuring that a point not in the original set exists in the circumcircle. This helps yield a better result by maximizing the smallest angles of the triangulation and avoid overly acute triangles.
{{< alert "lightbulb" >}}
You can find more information about [Delaunay Triangulation](https://en.wikipedia.org/wiki/Delaunay_triangulation) in this article.
{{< /alert >}}
There is a lot of geometry that goes into calculating the triangulation, presence of additional points, intersecting triangulations, and so on. So instead of reinventing the mathematical wheel (for now at least) I opted for a wonderful package by Fogleman, for Delaunay Triangulation, substantially speeding up my development of this project. I could of course sit down for hours and create my own, and down the line I may do it for the sake of gaining a deeper understanding of the math behind poly-fication, but frankly this was more just for fun.
{{< github repo="fogleman/delaunay" showThumbnail=false >}}
With this in place, creating the triangulations was a breeze. I started out by creating a set of points the Delaunay package required for triangulation, passed them in, and got my triangulations out. No flair, just simple and understandable.
```go
// Delaunay triangulation
delPoints := make([]delaunay.Point, len(points))
for i, p := range points {
  delPoints[i] = delaunay.Point{X: p[0], Y: p[1]}
}
tris, _ := delaunay.Triangulate(delPoints)
```
At this point, most of the complicated-sounding parts are done and we have a theoretical image/mesh that looks like this.
{{< figure 
  src="https://upload.wikimedia.org/wikipedia/commons/thumb/c/c4/Delaunay_Triangulation_%28100_Points%29.svg/375px-Delaunay_Triangulation_%28100_Points%29.svg.png"
  alt="A low poly mesh consisting of 100 points"
  caption="Illustration by [Inductiveload](https://commons.wikimedia.org/wiki/User:Inductiveload)"
>}}

The next step is was to sample the colors within the triangle, average them out, and draw them onto the output image. Firstly, to determine if the sampled pixel is within the triangle I used the barycentric coordinate system in order to quickly decide whether the pixel should be sampled or not. It works by mathematically determining the position of a point *P* with respect to a triangle *ABC*, calculating coefficients for *a, b, c* such that `a + b + c = 1` and `a*A + b*B + c*C = P`.
{{< alert "lightbulb" >}}
More info on [Barycentric Coordinate Systems](https://en.wikipedia.org/wiki/Barycentric_coordinate_system#Barycentric_coordinates_on_triangles) here!
{{< /alert >}}
The boiled down version looks a bit like this:
```go
/*
isPointInTriangle checks if a point (px, py) is inside the triangle defined by points (ax, ay), (bx, by), and (cx, cy).
It uses the barycentric coordinates method to determine if the point is within the triangle.
*/
func isPointInTriangle(px, py, ax, ay, bx, by, cx, cy float64) bool {
	p1x, p1y := cx-ax, cy-ay
	p2x, p2y := bx-ax, by-ay
	p3x, p3y := px-ax, py-ay

	dot0 := p1x*p1x + p1y*p1y
	dot1 := p1x*p2x + p1y*p2y
	dot2 := p1x*p3x + p1y*p3y
	dot3 := p2x*p2x + p2y*p2y
	dot4 := p2x*p3x + p2y*p3y

	denominator := (dot0 * dot3) - (dot1 * dot1)
	if denominator == 0 {
		return false
	}

	inverted := 1 / denominator
	u := ((dot3 * dot2) - (dot1 * dot4)) * inverted
	v := ((dot0 * dot4) - (dot1 * dot2)) * inverted
	return (u >= 0) && (v >= 0) && (u+v <= 1)
}
```

With the ability to quickly determine if the sampled point is inside the triangle or not, I can now start sampling colors within the triangle and returning the average value. This step is mentally easier to conceptualize; I start by getting the bottom left and top right of a rectangle that encompasses the triangle. For each pixel within that rectangle I check if it's within the triangle, and if it is I aggregate its RGBA value. Once iterations have concluded I take the sum and shift it to an RGBA-friendly value (0-255). Below this there is another helper function to fill the triangle which will take the returned average color, iterate through the rects again, and fill in the pixel with the new average color.
```go
/*
computeAverageColor computes the average color of the pixels inside the triangle defined by points (ax, ay), (bx, by), and (cx, cy).
It iterates over the bounding box of the triangle and checks if each pixel is inside the triangle
using the isPointInTriangle function.
*/
func computeAverageColor(img image.Image, ax, ay, bx, by, cx, cy float64) color.Color {
	minX := int(min(ax, min(bx, cx)))
	maxX := int(max(ax, max(bx, cx)))
	minY := int(min(ay, min(by, cy)))
	maxY := int(max(ay, max(by, cy)))

	var r, g, b, a, count uint32
	for y := minY; y <= maxY; y++ {
		for x := minX; x <= maxX; x++ {
			if isPointInTriangle(float64(x), float64(y), ax, ay, bx, by, cx, cy) {
				cr, cg, cb, ca := img.At(x, y).RGBA()
				r += cr
				g += cg
				b += cb
				a += ca
				count++
			}
		}
	}
	if count == 0 {
		return color.RGBA{0, 0, 0, 255}
	}
	return color.RGBA{
		uint8(r / count >> 8),
		uint8(g / count >> 8),
		uint8(b / count >> 8),
		uint8(a / count >> 8),
	}
}

// fillTriangle fills the triangle defined by points (ax, ay), (bx, by), and (cx, cy) in the image with the specified color.
func fillTriangle(img *image.RGBA, ax, ay, bx, by, cx, cy float64, col color.Color) {
	minX := int(min(ax, min(bx, cx)))
	maxX := int(max(ax, max(bx, cx)))
	minY := int(min(ay, min(by, cy)))
	maxY := int(max(ay, max(by, cy)))

	for y := minY; y <= maxY; y++ {
		for x := minX; x <= maxX; x++ {
			if isPointInTriangle(float64(x), float64(y), ax, ay, bx, by, cx, cy) {
				img.Set(x, y, col)
			}
		}
	}
}
```

Finally, all of these components could be wired together. I started by creating a new output image with. From there I then iterated over each of the triangulations provided by the Delaunay package, computed the average color, and filled the triangle.
```go
	out := image.NewRGBA(bounds)
	draw.Draw(out, bounds, img, bounds.Min, draw.Src)

	for i := 0; i < len(tris.Triangles); i += 3 {
		ia := tris.Triangles[i]
		ib := tris.Triangles[i+1]
		ic := tris.Triangles[i+2]

		ax, ay := tris.Points[ia].X, tris.Points[ia].Y
		bx, by := tris.Points[ib].X, tris.Points[ib].Y
		cx, cy := tris.Points[ic].X, tris.Points[ic].Y

		avgColor := computeAverageColor(img, ax, ay, bx, by, cx, cy)
		fillTriangle(out, ax, ay, bx, by, cx, cy, avgColor)
	}
```

## What I learned, Considerations, and Future Improvements
All in all this was a fun one-day project, it has been a good re-introduction to geometric maths for me and I now have a fun tool I can show off and perhaps have a use for in future projects. Naturally, I automated the build process for this project and the CLI tool is available in the repository releases. Some key takeaways I have are:

- I would not say this is a highly optimized tool. On static images it performs decently well - around 1.2 seconds for resizing a 1920x1920 image to 4K, however for gifs it does slow down quite a bit. When profiling the CPU using go's pprof tool it was clear the bottlenecks were `isPointInsideTriangle`, `computeAverageColor`, and `fillTriangle`.

- Some optimizations to research are ways to improve my version of barycentric coordinates, all three of the aforementioned methods relies on it. One solution could be to store the coordinates for each pixel of the triangle in a slice. It would have an increased memory overhead but potentially reduce the CPU usage.

- Once optimized further, I would like to extend this functionality to larger media formats like `mp4` and `mov`. Possibly even having the capability of low poly streaming.

- Adding more points by decreasing `density` will create more triangulations, this may be a good option to make dynamic based on the image width and height for a more consistent output; or as a CLI flag.

- One thing to consider when changing the `density` or `-intensity` flag of the tool is as density increases or the intensity decreases, the image will have a much blockier result. This may work in your favor if you want something much more abstract - such as if you want to test how much information we actually need to be able to identify an image.

- A progress bar in the CLI could be useful, especially for gifs and eventual video formats.

- This is a fun project, and I always welcome collaboration and ideas. If you would like to see new features or want to contribute you are more than welcome to open issues or pull requests in the repository.