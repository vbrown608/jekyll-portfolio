---
title: "Processing Animated Gifs with Imagemagick: A Tale of Two Ru's"
date: 2017-11-19T10:20:00Z
keywords: Rails 5.1, Carrierwave 1.2.1, Minimagick 4.8.0, Imagemagick 6.8.9-9
summary: How to scale, crop, and transform animated gifs on the Rails backend without artifacts
---

On EFF's [Security Education Companion](https://sec.eff.org) we scale and crop images on the backend to create thumbnails using Carrierwave and Minimagick. I noticed strange artifacts when we scaled animated gifs. In this post I'll explain how to process gifs cleanly with Minimagick. 

## tl;dr

* Gifs are often composed of partially-transparent layers that stack to create the illustion of a moving image.
* Some of the layers can be smaller than the overall image ("the canvas"). Scaling them individually changes their size relative to the canvas.
* It's therefore necessary to preprocess them so that each layer shows the full image before applying transformations.
* Preprocessing the image in that way makes the filesize much larger. It needs to be re-optimized after the transformation.
* Carrierwave's default `manipulate!` command doesn't allow that type of preprocessing. We need to call imagmagick's `convert` instead.

In code, it looks like this:

{% highlight ruby %}
version :thumbnail do
  process my_resize: 400
end

private
def gif_safe_transform!
  MiniMagick::Tool::Convert.new do |image| # Calls imagemagick's "convert" command
    image << @file.path
    image.coalesce # Remove optimizations so each layer shows the full image.

    yield image

    image.layers "Optimize" # Re-optimize the image.
    image << @file.path
  end
end

def my_resize(size)
  if @file.content_type == "image/gif"
    gif_safe_transform! do |image|
      image.resize "#{size}x#{size}" # Perform any transformations here.
    end
  else
    # Process other filetypes if necessary.
  end
end
{% endhighlight %}

## The problem

Here's a gif of RuPaul I used while testing this feature:

![Animation of Ru Paul in Drag saying "May the best woman win"](/images/best_woman.gif){:class='center'}

When I scaled the image down using imagemagick, I wound up with a mysterious second Ru!

![Animation of Ru Paul with a glitch where Ru appears overlaid over the original image](/images/glitchy_best_woman.gif){:class='center'}

## How are animated gifs stored?

Similar to a flipbook or a reel of film, gifs are comparised of a stack of frames called "layers" that are displayed one by one. Gifs have a key difference from a flipbook,though: layers can be partially transparent. They can also be smaller than the overall image, called the canvas. The metadata for each layer describes where it should be placed on the canvas when it's displayed.

Building gifs in this way makes it possible to create an animation by gradually stacking partially-transparent layers. Each layer overwrites only the part of the image that needs to change, creating the illusion of movement without the need to store the entire image at each frame. (1)

Here's a script letter "K" being drawn:

![A sample animation shows a script "K" being drawn](/images/script_k.gif)

This image is comprised of several smaller layers. At each frame, the next layer is added on top of the existing animation, gradually building up the complete "K".

![Small, partially-transparent layers stack on a larger canvas to create an script "K"](/images/script_k_frames.gif)

Running imagemagick's command line `identify` tool on the image reveals some metadata about each layer:

```bash
$ identify -format "%f canvas=%Wx%H size=%wx%h offset=%X%Y %D %Tcs\n" script_k.gif
script_k.gif canvas=53x54 size=1x1 offset=+0+0 None 8cs
script_k.gif canvas=53x54 size=19x15 offset=+1+7 None 4cs
script_k.gif canvas=53x54 size=16x18 offset=+17+5 None 4cs
script_k.gif canvas=53x54 size=17x23 offset=+8+21 None 4cs
script_k.gif canvas=53x54 size=5x6 offset=+5+42 None 20cs
script_k.gif canvas=53x54 size=16x10 offset=+34+8 None 4cs
script_k.gif canvas=53x54 size=17x12 offset=+19+15 None 4cs
script_k.gif canvas=53x54 size=11x14 offset=+14+23 None 4cs
script_k.gif canvas=53x54 size=14x12 offset=+21+31 None 4cs
script_k.gif canvas=53x54 size=11x10 offset=+31+38 None 4cs
script_k.gif canvas=53x54 size=5x6 offset=+39+44 None 200cs
```

We can see that each layer is smaller than the overall canvas. Each layer also includes an offset, which describes where to place the layer on the canvas when it's displayed.

Examining the RuPaul gif, we can see that layers where she's saying "And may the best woman" all look something like this:

![Animation of Ru Paul with a glitch where Ru appears overlaid over the original image](/images/adjoined_best_woman/frame_009.gif){:class='center'}
*__Frame 9__: canvas=400x225 size=265x400 offset=+244+0 None 0cs*{:class='caption'}

While the layers from the end of the gif ("win") look something like this:

![Animation of Ru Paul with a glitch where Ru appears overlaid over the original image](/images/adjoined_best_woman/frame_019.gif){:class='center'}
*__Frame 19__: canvas=400x225 size=400x224 offset=+0+0 None 0cs*{:class='caption'}

- Not all the layers are the same size
  - Show example from docs
  - Show output of inspect to demonstrate
  - When you scale/crop, each layer is scaled/cropped
  - Explain why that gives a certain output
- Coalesce
  - What it does - example from the docs
![Small, partially-transparent layers stack on a larger canvas to create an animation](/images/coalesce_k_montage.gif)
  - We need to use it - here's how
  - Problem: it causes the image size to get really big
    - Probably a footnote about why that other image got really *really* big
  - We need to re-optimize
- Applying all this in carrierwave/minimagick
  - "manipulate!" calls mogrify under the hood
  - mogrify processes batches of files in place, but it can't run the "layers" command
  - need to use "convert" instead - do it like this

![Animation of Ru Paul with a glitch where Ru appears overlaid over the original image](/images/scaled_best_woman.gif)

1. Not all gifs actually work this way. Each frame in a gif has a setting called "disposal". Disposal determines whether the preceding frame will be either (a) erased or (b) displayed underneath the current frame. The gifs described in this article all use a disposal setting of "None". That means the previous frame won't be erased - the current frame will be overlaid on top of it to create a composite image.
