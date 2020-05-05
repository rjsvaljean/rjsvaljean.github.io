---
title: "Image Explorer in Jupyter using Bokeh"
published: true
---

In the course of working on my masters thesis to automatically transcribe the writings of Scottish
folklorist Alexander Carmichael, I wrote a little utility to show me different pages of his field
notebooks in a Jupyter notebook to interact with this visual data in my notebook. So, broadly, the
goal was to be able to have a way to interactively select an image from a list and see it. Showing
all the images didn't work because too many and showing images by executing jupyter cells was
tedious.

It's a nice little example of using Bokeh (which I found a definition for in an [unrelated 
video](https://vimeo.com/channels/directorsseries/) about film which is nice) I'll show you the
code and then talk a bit about some of the design considerations and technical learnings from
writing it.

Code:

```
  1 from bokeh.models.widgets import Div
  2 from bokeh.models import Select, CustomJS
  3 from bokeh.io import output_notebook, show
  4 from bokeh.layouts import column
  5
...
  8 def image_selector(images_with_labels):
  9     """
 10     images_with_labels: dictionary with image_label as key and image_file_path
 11     """
 12     assert(len(images_with_labels) > 0)
 13     output_notebook()
 14
 15     image_div = Div(
 16         text="""<img src="./data/scans/UoEcar~1~1~23003~100001.jpg" alt="div_im
 17         width=700,
 18         height=700)
 19
 20     select = Select(title="Option:", value="foo", options=list(images_with_labe
 21     select.js_on_change(
 22         'value',
 23         CustomJS(
 24             args={
 25                 'image_div': image_div,
 26                 'images_with_labels': images_with_labels},
 27             code="""
 28             const image_location = images_with_labels[cb_obj.value]
 29             image_div.text = '<img src="' + image_location + '" alt="div_image"
 30             """))
 31
 32
 33     show(column(select, image_div))

```

The assertion on line 12 because I like using assertions as a type system when I don't have one.
Some documentation is sometimes better expressed programatically.

There are two Bokeh visual elements involved in this widget: 
- the `Div` on line 15 and
- the `Select` on line 20

The `Div` seems to be a container for arbitrary html and what I needed here was an image tag. So, I
used it. There's probably a more restrictive way to do this but I couldn't find it.

The `Select` is a bokeh model for a dropdown menu. But here's the interesting part: updating the Div
when the Select is interacted with.

So, there's two ways to do this from what I understood from a rushed glace at the docs. Either
running a server to execute python callbacks or writing your callbacks in javascript. I wanted to
keep this simple and using **two languages seemed the simpler choice than two processes**.

So, enter a third bokeh model: `CustomJS` (line 23) that allows you to bridge the python that 
generates the widget and the handler for interaction with that widget. In this case the handler was
simple: update the div with an image tag with a new source. Some messy string interpolation later:
voila. So, the handler logic is simple and the widget drawing logic is simple. The only tricky part
is bridging them. That's what lines 25-26 do. It makes the python references to the div and the
lookup dictionary that I use to store the list of images (paths keyed by image label) available to
the handler. Fortunately, bokeh's `Div` model and a python dictionary with string keys and values
are both encodable in javascript. You can update the `text` attribute of the `Div` and call
`__getitem__` on the dictionary as you would naturally do them in javascript which is fortunately
exactly like python.

One final element is combining the `Select` and `Div` vertically using the Bokeh's `column`  
layout model on line 33.

One final thing about the interface to all this. Using a dictionary to store the list of images
(with the key being some arbitrary label for the image and the value being the path to the image).
I used a list of image paths at first but changed to the dictionary because it gave be control over
what the images were called in the drop down (since the filenames were sometimes not descriptive
enough).
