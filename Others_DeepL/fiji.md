# Stitching with Fiji

Suppose we need to stitch such a dataset:

![dataset](/Others/imgstore/dataset.jpg)

Download 64-bit version: https://imagej.net/software/fiji/

![fiji](/Others/imgstore/fiji.jpg)

Plugins -> Stitching -> Grid/Collection stitching

Choose the order of the slides. Usually the slides are in a "snake" pattern:

|Snake from top to bottom|Snake from bottom to top|
|---|---|
|![order1](/Others/imgstore/order1.jpg)|![order2](/Others/imgstore/order2.jpg)|

Now we have to set it up:
- Number of slides by x (Grid size x)
- Number of rows (Grid size y)
- Tile overlap: are chosen experimentally 5-20%
- First file index i: If the name of the slide does not begin with 1
- Directory: slide folder
- File names for tiles: A pattern for the name of the slides. In our case: `{iiii}.jpg` (Slide names begin with 0001.jpg and on)
- Then there are some more obscure options

![options](/Others/imgstore/options.jpg)

After clicking on `Run` the program begins to witchcraft:

![magic](/Others/imgstore/magic.jpg)

And it will give you a picture:

![fused](/Others/imgstore/fused.jpg)

You can save the resulting picture via File -> Save As.
