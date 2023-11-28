---
title: heic to png
date: 2023-01-01 04:13:01 -0500
categories: [code samples]
tags: [ruby]     # TAG names should always be lowercase
---

Do you also share a small annoyance with .heic file types from iOS photos? Occassionally I'll take timestamp photos of items I want to sell on Reddit.
Most people generally upload their images to imgur and then share that link to Reddit. However, there's an additional step
of converting those heic images to png/jpg so that they'll be accepted by imgur. I've used a few of the free convert image websites but decided
to finally just build something I can run locally. 

With that, I discovered MiniMagick, a ruby wrapper for ImageMagick. 8 lines of code and voila

```ruby
require 'mini_magick'

Dir.glob('*.heic').each do |heic_file|
  output_png_file = File.basename(heic_file, '.heic') + '.png'

  MiniMagick::Tool::Convert.new do |convert|
    convert << heic_file
    convert << output_png_file
  end
end
```

It's currently configured to convert images in the same directory, however just pass in the dir of your choice to Dir.glob()

Alternatively, you can just install [ImageMagick](https://imagemagick.org/script/command-line-tools.php) directly. But where's the fun in that?
