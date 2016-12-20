---
title: F# and Dynamic Time Warping
date: 2016-12-13 21:07:21
tags:
- F#
- FSharp
- DTW
- Dynamic Time Warping
- EEG
- Signals
- Data
- Similarity
---

Today I look at using [F#](http://fsharp.org/) with the [NDtw](https://github.com/doblak/ndtw) package.  This is so I can play with some [dynamic time warping](https://en.wikipedia.org/wiki/Dynamic_time_warping).  In case you're not familar with DTW, the TLDR version is that it is a method to compare timeseries data that can differ in frequency.  This allows for a more nuanced data comparison that can capture shifted, compressed, and extended patterns.  It's a fun little algorithm to use and worth reading more about.

Onto the data. I've pulled an [EEG](https://www.kaggle.com/wanghaohan/eeg-brain-wave-for-confusion) dataset from Kaggle.  I've also included a copy [here](/data/EEG_data.zip) for posterity sake.  It contains EEG data of subjects watching short videos.  The goal of the dataset is mental state classification.  I won't be doing that here, but I can see using DTW as a method to facilitate classification based on channel smiliarties.

Using [Paket](https://github.com/fsprojects/Paket), here is a sample paket.dependencies file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget FSharp.Charting
nuget FSharp.Data
nuget NDtw
{% endcodeblock %}

Here is my standard boilerplate code, along with the VideoId that I will be using for testing.  As a note, all of the data columns are floats, including the subject and video ids.  If I was doing something more serious I'd be inclined to convert these, but do do something quick I'll deal with it.

{% codeblock lang:fsharp %}
#r "../packages/FSharp.Charting/lib/net40/FSharp.Charting.dll"
#r "../packages/FSharp.Data/lib/net40/FSharp.Data.dll"
#r "../packages/ndtw/lib/net40/ndtw.dll"
#load "../packages/FSharp.Charting/FSharp.Charting.fsx"

open System
open System.IO
open FSharp.Charting
open FSharp.Data
open NDtw

// Limit to only video 6
let videoIdFilter = 6. 
{% endcodeblock %}

Load data using a type provider.  Since the file doesn't have headers, I'll use Schema to define the column names.  As a note, the ```[<Literal>]``` attribute on eegDataFile is so I can use the string in the CsvProvider.

{% codeblock lang:fsharp %}
[<Literal>]
let eegDataFile = "..\\data\\eeg\\eeg_data.csv"

type EegData = CsvProvider<eegDataFile, HasHeaders = false, Schema = "SubjectID,VideoID,Attention,Mediation,RawEEG,Delta,Theta,Alpha1,Alpha2,Beta1,Beta2,Gamma1,Gamma2,PredefinedLabel,UserDefinedLabel">

let fileData = EegData.Load eegDataFile
let allData = fileData.Rows 

let subjectIds = fileData.Rows |> Seq.map (fun x -> x.SubjectID) |> Seq.distinct 
{% endcodeblock %}


I now create a function to extract subject and video specific rows from the dataset.  I also reduce the dataset to a single value for the signal.  I've decided to use the Theta channel.  This is arbitrary, but is primarily due to this quote from the dataset's Kaggle page "Past research has indicated that Theta signal is correlated with confusion level."  This leads me to believe it might be the most likely channel to find interesting comparisons.  So the resulting charts are easier to read, I limit the dataset to the first 100 rows of data per subject/video combination.

{% codeblock lang:fsharp %}
// Get subject & video specific data, only pull the first 100 records of the dataset
let dataset subjectId videoId = 
    allData
    |> Seq.filter(fun x -> x.SubjectID = subjectId && x.VideoID = videoId)
    |> Seq.map (fun x -> x.Theta) 
    |> Seq.take 100
    |> Seq.toList
{% endcodeblock %}

Here is the dynamic time warping distance calculation function.  The call is straight foward.  After all this setup its almost anticlimatic. It expects 2 parameters, both being an array of values. 

{% codeblock lang:fsharp %}
// Get the distance between two signals
let distance (a:float seq) (b:float seq) = 
    let dtw = new Dtw(
        a |> Seq.toArray,
        b |> Seq.toArray)
    dtw.GetCost()
{% endcodeblock %}


The NDtw library allows for a more complicated DTW call if so desired.  I've made an alternate distance function using the more complex version.  As an example, it allows for different distance calculations: Euclidean, Manhattan, Maximum, SquaredEuclidean. It also allows for limitations how much the path diverges from the standard path.

{% codeblock lang:fsharp %}
// Get the distance between two signals (using more advanced dtw call)
let distanceAdvanced (a:float seq) (b:float seq) = 
    let dtw = new Dtw(
        a |> Seq.toArray,
        b |> Seq.toArray,
        DistanceMeasure.Euclidean,
        true, // boundaryConstraintStart 
        true, // boundaryConstraintEnd
        Nullable<int>(2), // slopeStepSizeDiagonal
        Nullable<int>(2), // slopStepSizeAside
        Nullable<int>(2)) // sakoeChibaMaxShift
    dtw.GetCost()
{% endcodeblock %}

As a short aside, the library offers a couple other bits of useful functionality that I am not using right now, but worth mentioning.

{% codeblock lang:fsharp %}
// Example comparison of 2 small datasets
let exampleDtw = new Dtw(
    [| 1.; 3.; 7.; 8.; 11.;  6.; 6.; |],
    [| 3.; 6.; 3.; 7.; 13.; 12.; 7.; |])


// Get the coordinate path used for the best match between the datasets
exampleDtw.GetPath()

val it : (int * int) [] =
  [|(0, 0); (1, 1); (1, 2); (2, 3); (3, 3); (4, 4); (4, 5); (5, 6); (6, 6)|]


// Get the matrix of distance comparisons on a point by point basis
exampleDtw.GetDistanceMatrix()

val it : float [] [] =
  [|[|2.0; 5.0; 2.0; 6.0; 12.0; 11.0; 6.0; 0.0|];
    [|0.0; 3.0; 0.0; 4.0; 10.0; 9.0; 4.0; 0.0|];
    [|4.0; 1.0; 4.0; 0.0; 6.0; 5.0; 0.0; 0.0|];
    [|5.0; 2.0; 5.0; 1.0; 5.0; 4.0; 1.0; 0.0|];
    [|8.0; 5.0; 8.0; 4.0; 2.0; 1.0; 4.0; 0.0|];
    [|3.0; 0.0; 3.0; 1.0; 7.0; 6.0; 1.0; 0.0|];
    [|3.0; 0.0; 3.0; 1.0; 7.0; 6.0; 1.0; 0.0|];
    [|0.0; 0.0; 0.0; 0.0; 0.0; 0.0; 0.0; 0.0|]|]


// Get the matrix of cost (aka difference) between the datasets 
exampleDtw.GetCostMatrix()

val it : float [] [] =
  [|[|11.0; 11.0; 8.0; 16.0; 28.0; 22.0; 17.0; infinity|];
    [|9.0; 9.0; 6.0; 10.0; 22.0; 16.0; 11.0; infinity|];
    [|15.0; 11.0; 10.0; 6.0; 13.0; 12.0; 7.0; infinity|];
    [|18.0; 13.0; 11.0; 6.0; 8.0; 7.0; 7.0; infinity|];
    [|26.0; 22.0; 17.0; 9.0; 5.0; 3.0; 6.0; infinity|];
    [|21.0; 18.0; 18.0; 15.0; 14.0; 7.0; 2.0; infinity|];
    [|21.0; 18.0; 18.0; 15.0; 14.0; 7.0; 1.0; infinity|];
    [|infinity; infinity; infinity; infinity; infinity; infinity; infinity; infinity|]|]


// Get difference between datasets
exampleDtw.GetCost()

val it : float = 11.0
{% endcodeblock %}


Back to the task at hand, I build a function to compare each subject's signal against each other subject's signal.  Then I get the best match I can find and store the results in subject1, subject2, and difference.


{% codeblock lang:fsharp %}
// Compare each subject against every other subject
let comparisons = 
    subjectIds
    |> Seq.map (fun x -> 
        subjectIds
        |> Seq.filter (fun y -> y <> x)
        |> Seq.map (fun y -> 
            let d = distance (dataset x videoIdFilter) (dataset y videoIdFilter)
            (x, y, d)))
    |> Seq.concat
    |> Seq.sortBy (fun (_, _, d) -> d)

let (subject1, subject2, difference) = 
    comparisons
    |> Seq.take 1
    |> Seq.exactlyOne
{% endcodeblock %}

For visualization purposes, I create a comparison chart for each subject against Subject 1.  Then I save the charts to files.

{% codeblock lang:fsharp %}
Chart.Combine([
    Chart.FastLine(dataset subject1 videoIdFilter, Name=sprintf "Subject %1.0f" subject1).WithLegend(true);
    Chart.FastLine(dataset subject2 videoIdFilter, Name=sprintf "Subject %1.0f" subject2).WithLegend(true)])
|> Chart.WithTitle(String.Format("Difference: {0:n1}", difference))
|> Chart.Show

// Show comparison charts
comparisons
// Only include matches against subject1
|> Seq.filter (fun (s1, s2, d) -> s1 = subject1)
// Exclude subject1 vs. subject1
|> Seq.filter (fun (s1, s2, d) -> s2 <> subject1)
// Chart each pair
|> Seq.iter (fun (s1, s2, d) -> 
    Chart.Combine([
        Chart.FastLine(dataset s1 videoIdFilter, Name=sprintf "Subject %1.0f" s1).WithLegend(true);
        Chart.FastLine(dataset s2 videoIdFilter, Name=sprintf "Subject %1.0f" s2).WithLegend(true)])
    |> Chart.WithTitle(String.Format("Difference: {0:n1})", d))
    |> Chart.WithYAxis(Min = 0., Max = 1000000.)
    |> Chart.Save (Path.Combine(__SOURCE_DIRECTORY__, String.Format(@"..\\data\\eeg\\chart_{0:n0}_{1:n0}.png", s1, s2)))
    //|> Chart.Show
    )
{% endcodeblock %}

Here are the top couple matches, in the format (subject1, subject2, distance).

{% codeblock lang:fsharp %}
// Show distances for our winner 
comparisons
|> Seq.filter (fun (s1, _, _) -> s1 = 1.0)
|> Seq.sortBy (fun (_, _, d) -> d)
|> Seq.iter (fun (s1, s2, d) -> printfn "%1.0f %1.0f %10.1f" s1 s2 d)

1 7  4442366.0
1 4  5256824.0
1 2  6801045.0
1 9  9462037.0
1 8 11714938.0
1 0 12763188.0
...
{% endcodeblock %}

Here is the top signal match.

![Best Match](/images/dtw1/chart_1_7.png)

Here are the other subjects compared to subject 1.

![Comparison: Signal 1 and Signal 4](/images/dtw1/chart_1_4.png)

![Comparison: Signal 1 and Signal 2](/images/dtw1/chart_1_2.png)

![Comparison: Signal 1 and Signal 9](/images/dtw1/chart_1_9.png)

![Comparison: Signal 1 and Signal 8](/images/dtw1/chart_1_8.png)

![Comparison: Signal 1 and Signal 0](/images/dtw1/chart_1_0.png)

![Comparison: Signal 1 and Signal 6](/images/dtw1/chart_1_6.png)

![Comparison: Signal 1 and Signal 3](/images/dtw1/chart_1_3.png)

![Comparison: Signal 1 and Signal 5](/images/dtw1/chart_1_5.png)

There is it.  The best match it can find is between subjects 1 and 7, although 1 and 4 are a close second. This has been a fun experiment.  Until next time.



