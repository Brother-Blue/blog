---
title: Improvements from Previous Observations
slug: low-poly-improvements
description: |
  Improving the Low Poly tool and optimizations, further analysis and review. More considerations and future plans.
date: '2025-06-30T15:56:10-04:00'
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
series_order: 2
---

## Introduction

The project has come along nicely the past week. After reviewing the considerations from the first iteration, and ironing out some kinks in my workflows, I am now at a point where I can comfortably and consistently work on this project and release new versions without any troubles. A majority of the updates were to quash some overlooked bugs and make necessary changes to make video file support more enjoyable. I'm always hesitant to pull the *optimizations* cards when explaining changes, as a former colleague had once told me *"One of the biggest hurdles you can give yourself is trying to optimize your project/code too early"*; however, I think they'll forgive me this time.

All of the code is available at my GitHub.
{{< github repo="brother-blue/low-poly-converter" showThumbnail=false >}}

## Quality of Life Changes

- I added in a `-debug` flag to the CLI tool, this was to get more help from the `pprof` tool as I explain in the next section.
- I also added in a `-showProgress` flag which will add a progress bar. This was a nice change when running the tool against larger GIFs to make sure it wasn't hanging.

## The Problem Part 2

The first iteration was designed purely for static images, and for the most part it worked pretty well. What worked less well was when I wanted to add a feature to process GIFs. I grabbed a relatively small GIF, approximately 480p and 36 frames, with resizing and applying low poly it could take upwards of **four** minutes. This was not ideal seeing that the entire GIF was just under two seconds long. Using the newer debug feature in the tool and passing it along to Go's `pprof` tool, I was quickly able to visualize and see where the bottleneck was - it was also quite obvious where it was since this project only had a few components - but it always pays to be certain. There was a glaring red, thick line that went straight to the barycentric coordinate check and the color sampling. 

## Finding a Solution

As a rudimentary test, instead of calling the barycentric check for the color sampling and again for the color filling, I stored the pixels that were within the triangle into a slice and passed those into the successive methods that relied on knowing those points. There was an immediate improvement of about 30 seconds, not great but definitely a step in the right direction. The next step to improving was one I had had not much hands-on experience with in the past using Go, the splendid `goroutine`.

Instead of processing each frame sequentially, they would now be processed in parallel as shown below. This function takes in all of the images of the GIF and starts a goroutine to process each one individually. When completed, they are recombined and the resulting GIF is returned.
```go
func ProcessGifParallel(images *gif.GIF, width, height, intensity int, bar *progressbar.ProgressBar) *gif.GIF {
	var wg sync.WaitGroup
	frames := make([]*image.Paletted, len(images.Image))

  // If the width and height are provided
  // then the GIF config needs to be updated
	w, h := images.Config.Width, images.Config.Height
	if width > 0 && height > 0 {
		w, h = width, height
	}
	images.Config.Width = w
	images.Config.Height = h

	for idx, frame := range images.Image {
		wg.Add(1)
    // Run each frame process on its own gorouting
		go func(idx int, frame *image.Paletted) {
			defer wg.Done()
			rgba := image.NewRGBA(frame.Bounds())
			draw.Draw(rgba, frame.Bounds(), frame, image.Point{}, draw.Src)
			if width > 0 && height > 0 {
				rgba = ResizeImage(rgba, width, height).(*image.RGBA)
			}

			processed := ApplyLowPoly(rgba, intensity)
			bounds := image.Rect(0, 0, w, h)
			paletted := image.NewPaletted(bounds, frame.Palette)
			draw.Draw(paletted, bounds, processed, processed.Bounds().Min, draw.Over)
			frames[idx] = paletted

			if bar != nil {
				bar.Add(1)
			}
		}(idx, frame)
	}
	wg.Wait()
	images.Image = frames
	return images
}
```
I had concerns about race conditions initially, but after some quick research it seemed to not be a concern I needed as each goroutine was working on its own frame. After some refactoring and reconnecting, it worked without any headaches. Checking the CPU and memory profiles, I thought I had made a mistake somewhere because it had showed now 200-300 seconds - but the total duration was 14 seconds. This was just due to my misinterpretation of pprof, and after doing some math to find the calculations *per frame*, the processing speed had gone from six seconds per frame to now 0.402 (402ms) per frame. Much better, but still not where I need it.

I squeezed a bit more out, but the changes were hardly noticeable and I was teatering on the edge of optimizing my code too early. A good note for the readers, when you hit this plateau, it's usually a good time to find another road.
{{< alert icon="circle-info" cardColor="#0077b6" iconColor="#CAF0F8" >}}
This does not always hold true, especially at scale. For example a 2ms improvement may not seem like much, but if you consider a service like Cloudflare that is a massive improvement.
{{< /alert >}}
My service luckily is not Cloudflare, so I'm not going to stress over 400ms per frame - because I know what my next mountain to climb is.

## Finding Another Problem
As of now, the project is using the standard `image` package from Go, and up until now it has worked great. But it has also become my new bottleneck - and I'm not going to say it is 100% because of this, there is probably a good bit of sloppy setup on my end as well, but the profiler does show a substantial amount of CPU time being used on converting between RGBA frames to Paletted frames. This happens when the GIF is first loaded in and fanned out, each frame has a new RGBA frame created for faster color manipulation which later has the low poly filter applied. The processed RGBA image is then drawn back as a paletted image, recognizeable by the `gif` package.

{{< alert icon="lightbulb" >}}
If you'd like to read more about the standard image/gif packages you can find them [here!](https://pkg.go.dev/image)
{{< /alert >}}

Mountain, meet Chris.

## Takeaways, Considerations, and Feature Plans

At this stage, most of my takeaways are technical rather than theoretical. The most immediate improvement I’m eyeing is to ensure all pixel values are cast to integers. Not only does this make sense for image data, but it also has a real impact on memory usage: currently, each pixel is a `float64`, but switching to `int` could potentially halve the memory footprint—something that will definitely matter at scale.

It’s also worth acknowledging that there are plenty of mature packages and tools out there, built by people with deep expertise in image processing. But as I mentioned in my previous post, this project is about learning and having fun. I’m committed to pushing the limits of what I can do with Go’s standard package for now. If I hit another wall, maybe I’ll take a shot at building my own package from scratch.

### Things to Improve
- Explore building a custom image package, especially for GIF and/or video formats.
- Revisit and optimize my use of data structures for better performance and clarity.

### Just for Fun
- Add a flag to skip the poly-fication step, enabling a quick, on-the-fly resizing CLI.

As always if you have any feedback or ideas, feel free to drop by the repository!