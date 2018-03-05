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
* It's therefore necessary to preprocess them so that each layer show the full-sized image at that point in the animation before applying transformations.
* Preprocessing the image in that way makes the filesize much larger. It needs to be re-optimized after the transformation.
* Carrierwave's default `manipulate!` command doesn't allow that type of preprocessing. We need to call imagmagick's `convert` directly instead.

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

Building gifs in this way makes it possible to create an animation by gradually stacking partially-transparent layers. Each layer overwrites only the part of the image that needs to change, creating the illusion of movement without the need to store the entire image at each frame[^1].

Here's a script letter "K" being drawn[^2]:

![A sample animation shows a script "K" being drawn](/images/script_k.gif)

This image is comprised of many small layers. At each frame, the next layer is added on top of the existing animation, gradually building up the complete "K".  We can see that each layer is smaller than the overall canvas. Each layer also includes an offset, which describes where to place the layer on the canvas when it's displayed.

![Small, partially-transparent layers stack on a larger canvas to create an script "K"](/images/script_k_frames.gif)

ImageMagick's command line `identify` tool outputs some metadata about each layer of the gif.

{% highlight shell_session %}
$ identify -format "%f canvas=%Wx%H size=%wx%h offset=%X%Y %D %Tcs\n" best_woman.gif
{% endhighlight %}

Here's a layer that's representative of the first half of the gif. Ru is saying, "And may the best woman". The original layer is narrower than the overall canvas. It's displayed at an offset so that it's centered within the canvas. When we resize the image, we resize each layer without considering its original size. As a result, these layer become far too wide[^3]. They keep their original offset, bumping Ru over to the right.

<div class="flex-column-wrapper">
<div class="left-col" markdown="1">
![](/images/best_woman_09.gif){:class='center'}
*__Frame 9 before__: canvas=500x281 size=186x281 offset=+171+0 None 0cs*{:class='caption'}
</div>

<div class="right-col" markdown="1">
![](/images/glitchy_best_woman_09.gif){:class='center'}
*__Frame 9 after__: canvas=400x225 size=265x400 offset=+244+0 None 0cs*{:class='caption'}
</div>
</div>

Frame 19 is representative of the second half of the gif (where Ru says "win"). Those frames are the same size as the overall canvas, so there's no need to display them at an offset. When we resize the image, they keep their original dimensions and position relative to the canvas.

<div class="flex-column-wrapper">
<div class="left-col" markdown="1">
![](/images/adjoined_best_woman/frame_019.gif){:class='center'}
*__Frame 19 before__: canvas=500x281 size=500x280 offset=+0+0 None 0cs*{:class='caption'}
</div>

<div class="right-col" markdown="1">
![](/images/glitchy_best_woman_19.gif){:class='center'}
*__Frame 19 after__: canvas=400x225 size=400x224 offset=+0+0 None 0cs*{:class='caption'}
</div>
</div>

## Coalesce

To avoid these artifacts, we need to do some preprocessing on each layer of the gif before we transform it. We use an imagemagick command called *coalesce*.

Coalesce converts each layer of the image into a full-canvas snapshot showing what the gif should look like at that moment in the animation. For the example K animation, coalesce would turn each layer into a full-size, partially completed K:

![Montage of gif layers showing the output of the coalesce command. Each layer now shows the full gif at that point in the animation.](/images/coalesce_k_montage.gif)

`convert best_woman.gif -coalesce best_woman_coalesced.gif` gives us a coalesced version of the original image.

An unsurprising side effect of coalesce is that it makes the filesize of the gif much larger. Where previously each frame had to show the pixels that changed at the moment, each frame now must show the entire image. In the case of lengthy animated gifs where very little changes from frame to frame, the file size can grow enormously.

For that reason, after running coalesce and then transforming the image, we need to reoptimize the gif.

The final imagemagick command will be something like ` convert best_woman.gif -coalesce -resize {width}x{height} -layers Optimize`.

## Calling it in Rails

The next step is to translate the command line imagemagick command into Ruby code that will be executed by CarrierWave.

CarrierWave normally uses the `manipulate!` block to build commands to imagemagick. `manipulate!` called imagemagick's `mogrify` command under the hood. `mogrify` is similar to `convert`, and accepts many of the same arguements. It's used to modify batches of files in place. Unfortunately, it doesn't support the `-layers` command, which is required to both coalesce[^4] and optimize the gif.

We can reach into MiniMagick, the recommended Ruby wrapper for imagemagick, and call convert as `MiniMagick::Tool::Convert.new`. Then we build up the command in the usual way. Unlike `mogrify`, `convert` doesn't overwrite the original final, so we need to finish by passing an output path.

Here, again, is the Ruby-fied call to `convert`:

{% highlight ruby %}
MiniMagick::Tool::Convert.new do |image| # Calls imagemagick's "convert" command
  image << @file.path
  image.coalesce # Remove optimizations so each layer shows the full image.

  yield image # Caller applies the transformation.

  image.layers "Optimize" # Re-optimize the image.
  image << @file.path
end
{% endhighlight %}

Which gives us the correctly resized image.
![Correctly resized gif of RuPaul with no weird artifacts](/images/scaled_best_woman.gif){:class='center'}

<hr/>

[^1]: Not all gifs actually work this way. Each frame in a gif has a setting called "disposal". Disposal determines whether the preceding frame will be either (a) erased or (b) displayed underneath the current frame. The gifs described in this article all use a disposal setting of "None". That means the previous frame won't be erased - the current frame will be overlaid on top of it to create a composite image.

[^2]: This example, as well as most of my understanding of the gif format, comes from Anthony Thyssen's excellent guide at [https://www.imagemagick.org/Usage/anim_basics/](https://www.imagemagick.org/Usage/anim_basics/).

[^3]: In this case, the each frame grows to be 400px in its' largest dimension.

[^4]: `-coalesce` is shorthand for `-layers coalesce`.

