---
title: Face Detection with F# and Accord.NET
date: 2016-11-21 09:48:50
tags:
- Machine Learning
- F#
- Accord.NET
- Computer Vision
---

This is a quick experiment using [F#](http://fsharp.org/) and [Accord.NET](http://accord-framework.net/) to do face detection.  The method uses the provided Haar-like feature detection.  The results aren't particularly good, but for little effort it's an ok start.  At a minimum, it does reasonably well at detecting potential regions of interest.  For the test images, the best improvements were found when constraining the min/max range based on the known sizes of faces in the pictures.

Using [Paket](https://github.com/fsprojects/Paket), here is a sample ```paket.dependencies``` file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget Accord
nuget Accord.Math
nuget Accord.Statistics
nuget Accord.MachineLearning
nuget Accord.Vision
nuget Accord.Imaging
{% endcodeblock %}

This is the boring setup stuff.  

{% codeblock lang:fsharp %}
// ref: http://accord-framework.net/docs/html/N_Accord_Vision_Detection.htm

#i "../packages"
#r "../packages/Accord/lib/net45/accord.dll"
#r "../packages/Accord.Imaging/lib/net45/accord.imaging.dll"
#r "../packages/Accord.Vision/lib/net45/accord.vision.dll"

open System
open Accord
open Accord.Imaging
open Accord.Vision
open Accord.Vision.Detection
open System.Drawing
open System.Drawing.Imaging
open System.IO

let imageRoot = Path.GetFullPath(Path.Combine(__SOURCE_DIRECTORY__, "../data/haar_face/"))
let resultsRoot = Path.GetFullPath(Path.Combine(__SOURCE_DIRECTORY__, "../data/haar_face/results/"))
{% endcodeblock %}

Two cascades are provided, ```FaceHaarCascade()``` and ```NoseHaarCascade()```.  Custom ones can be created, but for an initial test, one of the provided cascades is good enough.  The minSize and maxSize values are hardcoded hacks to match the expected face sizes in the test images.  They are used to define the minimum and maximum window size to consider as the algorithm scans the image.

{% codeblock lang:fsharp %}
let cascade = Cascades.FaceHaarCascade()
//let cascade = Cascades.NoseHaarCascade()
let minSize = 200 
let maxSize = 2000
{% endcodeblock %}

This is a function that will add a bounding box to the bitmap with the specified line color and width.

{% codeblock lang:fsharp %}
// Draw bound-box on Bitmap 
// TopLeft: (x1, y1)
// BottomRight: (x2, y2) 
let drawRectangle (bitmap:Bitmap) (x1:int) (y1:int) (x2:int) (y2:int) (lineWidth:int) (lineColor:Color) = 
    [x1..x2] 
    |> List.iter (fun x ->
        [0..lineWidth] 
        |> List.iter (fun i -> 
            bitmap.SetPixel(x, y1 + i, lineColor) 
            bitmap.SetPixel(x, y2 - i, lineColor)))

    [y1..y2] 
    |> List.iter (fun y -> 
        [0..lineWidth] 
        |> List.iter (fun i -> 
            bitmap.SetPixel(x1 + i, y, lineColor) 
            bitmap.SetPixel(x2 - i, y, lineColor)))
{% endcodeblock %}

This is the core component of interest.  It loads a bitmap, runs the object detector, draws bounding boxes around the detected locations, then saves a "result" image.

{% codeblock lang:fsharp %}
let processImage (cascade:HaarCascade) (minSize:int) (maxSize:int) (resultsDir:string) (imageName:string) =
    let resultImageName = Path.Combine(resultsDir, Path.GetFileName(imageName))        
    let bitmap = new Bitmap(imageName)

    let haar = HaarObjectDetector(cascade, minSize)
    haar.MaxSize <- new Size(new Point(new Size(maxSize, maxSize)))
    let faceFinder = haar.ProcessFrame(bitmap)

    faceFinder
    |> Array.iter (fun r -> 
        drawRectangle bitmap r.X r.Y (r.X + r.Width) (r.Y + r.Height) 5 Color.Blue)

    File.Delete resultImageName
    bitmap.Save(resultImageName)
{% endcodeblock %}

The below code gets a list of qualifying images, then sends them through the processing function.

{% codeblock lang:fsharp %}
let isImageFile (fileName:string) = 
    fileName.EndsWith(".jpg", StringComparison.OrdinalIgnoreCase)
    || fileName.EndsWith(".png", StringComparison.OrdinalIgnoreCase)

let imageNames = 
    Directory.GetFiles imageRoot 
    |> Array.filter isImageFile

imageNames 
|> Array.iter (processImage cascade minSize maxSize resultsRoot)
{% endcodeblock %}

That is all there is to it.  Certainly the method can be improved, but hopefully this shows a small taste of what can be done with F#.

