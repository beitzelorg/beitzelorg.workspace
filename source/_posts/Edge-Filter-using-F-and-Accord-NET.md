---
title: 'Edge Filter using F# and Accord.NET'
date: 2016-11-23 07:56:08
tags:
- F#
- Accord.NET
- Images
---

This is a sample of how to apply edge filters to a set of images using [F#](http://fsharp.org/) and [Accord.NET](http://accord-framework.net/).  The framework provides several [Filters](http://accord-framework.net/docs/html/N_Accord_Imaging_Filters.htm) for image manipulating.  Since I'm interested in edge enhancement I'll limit my scope to those filters.  In particular I've selected the ```DifferenceEdgeDetector()```.

Using [Paket](https://github.com/fsprojects/Paket), here is a sample ```paket.dependencies``` file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget Accord
nuget Accord.Imaging
{% endcodeblock %}

This is the boring setup stuff.  

{% codeblock lang:fsharp %}
// Ref: http://accord-framework.net/docs/html/N_Accord_Imaging_Filters.htm

#i "../packages"
#r "../packages/Accord/lib/net45/accord.dll"
#r "../packages/Accord.Imaging/lib/net45/accord.imaging.dll"

open System
open System.Drawing
open System.Drawing.Imaging
open System.IO
open Accord
open Accord.Imaging
open Accord.Imaging.Filters

let imageRoot = Path.GetFullPath(Path.Combine(__SOURCE_DIRECTORY__, "../data/"))
let resultsRoot = Path.GetFullPath(Path.Combine(__SOURCE_DIRECTORY__, "../data/results/"))
{% endcodeblock %}

Loading and applying the filter is straight-forward.  The only additional point worthy of mention is that most of the edge filters require the image to be in grayscale.  That conversion is included in this function.

{% codeblock lang:fsharp %}
// Process image and save a result file with edge filter applied
let filterImage (resultsDir:string) (imageName:string) =
    let resultImageName = Path.Combine(resultsDir, Path.GetFileName(imageName))        
    let bitmap = new Bitmap(imageName)

    // Need to reduce to grayscale, filter needs a reduced color bitmap to process
    let bitmapGray = Grayscale.CommonAlgorithms.BT709.Apply(bitmap)
    let filter = new DifferenceEdgeDetector()

    filter.ApplyInPlace(bitmapGray)

    File.Delete resultImageName
    bitmapGray.Save(resultImageName)
{% endcodeblock %}

The below code gets a list of qualifying images, then sends them through the filtering function.

{% codeblock lang:fsharp %}
// Is file an image file?
let isImageFile (fileName:string) = 
    fileName.EndsWith(".jpg", StringComparison.OrdinalIgnoreCase)
    || fileName.EndsWith(".png", StringComparison.OrdinalIgnoreCase)

// Get Image list to process
let imageNames = 
    Directory.GetFiles imageRoot 
    |> Array.filter isImageFile

// Process images
imageNames 
|> Array.iter (filterImage resultsRoot)
{% endcodeblock %}

Below is an example of the edge filter applied.

![Before](/images/edges_before.jpg)

![After](/images/edges_after.jpg)

