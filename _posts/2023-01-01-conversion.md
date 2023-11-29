---
title: heic to png
date: 2023-01-01 04:13:01 -0500
categories: [utilities]
tags: [ruby]
img_path: /assets/img/
---

Do you also share a small annoyance with .heic file types from iOS photos? Occassionally I'll take timestamp photos of items I want to sell on Reddit.
I would typically upload my images to imgur and then share that link to Reddit. However, imgur doesn't support heic so I'm burdened with
converting those heic images to png/jpg. I've used a few of the free convert image websites but decided
to finally just build something I can run locally. 

With that, I discovered MiniMagick, a ruby wrapper for ImageMagick. a few lines of code and voila!

```ruby
begin
  require 'mini_magick'
rescue LoadError
  puts "The mini magick gem is required to run this script. Please run the following: "
  puts "gem install mini_magick"
end

heic_files = Dir.glob('*.{heic, HEIC}')
heic_files.each do |heic_file|
  output_png_file = File.basename(heic_file, '.*') + '.png'
  puts "Converting #{heic_file} to png..."
  MiniMagick::Tool::Convert.new do |convert|
    convert << heic_file
    convert << output_png_file
  end
  puts "Conversion complete!"
end
```

Taking it one step further, let's make this tool available from anywhere in my shell. First I'll configure an alias in my `.zshrc` config.

```bash
alias conv='ruby -/scripts/heic_to_png.rb'
```

Now I can run my conv command from my terminal in any directory and it will begin processing conversions! 

![Conversion Complete](img_convert.png)

Alternatively, you can just install [ImageMagick](https://imagemagick.org/script/command-line-tools.php) directly. But where's the fun in that?
