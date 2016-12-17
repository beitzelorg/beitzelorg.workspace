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

Today I look at using [F#](http://fsharp.org/) with the [NDtw](https://github.com/doblak/ndtw) package.  This is so I can play with some [Dynamic_time_warping](https://en.wikipedia.org/wiki/Dynamic_time_warping).  If you're not familar with it, 

[EEG](https://www.kaggle.com/wanghaohan/eeg-brain-wave-for-confusion)


Using [Paket](https://github.com/fsprojects/Paket), here is a sample paket.dependencies file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget FSharp.Charting
nuget NDtw
{% endcodeblock %}

Here is some of the basic setup stuff.

{% codeblock lang:fsharp %}
#r "../packages/FSharp.Charting/lib/net40/FSharp.Charting.dll"
#r "../packages/ndtw/lib/net40/ndtw.dll"
#load "../packages/FSharp.Charting/FSharp.Charting.fsx"

open System
open System.IO
open FSharp.Charting
open NDtw

// Limit to only video 3
let videoIdFilter = 3.
{% endcodeblock %}

Load data from the csv file.

{% codeblock lang:fsharp %}
let allData = 
    Path.Combine(__SOURCE_DIRECTORY__, @"..\\data\\eeg\\eeg_data.csv")
    |> File.ReadLines
    |> Seq.map (fun x ->x.Split ',')
    |> Seq.map (fun x -> Array.map (fun y -> float(y)) x)
    |> Seq.toList
{% endcodeblock %}

I now create a function to extract subject and video specific rows from the dataset.  Column 0 is the UserId, Column 1 is the VideoId, and Column 5 is the Raw EEG data.  I also limit the dataset to 100 records.  I only do this to make the resulting charts easier to read. 

{% codeblock lang:fsharp %}
// col0 = subject, col1=video, column5=raw eeg
let dataset subjectId videoId = 
    allData
    |> List.filter(fun x -> x.[0] = subjectId && x.[1] = videoId)
    |> List.map (fun x -> x.[5]) 
    |> List.take 100
{% endcodeblock %}

Here is the dynamic time warping distance calculation function.  The call is straight foward.  

{% codeblock lang:fsharp %}
let distance (a:float list) (b:float list) = 
    let dtw = new Dtw(
        a |> List.toArray,
        b |> List.toArray)
    dtw.GetCost()
{% endcodeblock %}

The NDtw library allows for a more complicated DTW call if so desired.  I've made an alternate distance function using the more complex version.

{% codeblock lang:fsharp %}
let distance2 (a:float list) (b:float list) = 
    let dtw = new Dtw(
        a |> List.toArray,
        b |> List.toArray,
        DistanceMeasure.Euclidean,
        true, // boundaryConstraintStart 
        true, // boundaryConstraintEnd
        Nullable<int>(2), // slopeStepSizeDiagonal
        Nullable<int>(2), // slopStepSizeAside
        Nullable<int>(2)) // sakoeChibaMaxShift
    dtw.GetCost()

(*
Possible distance measures: 
DistanceMeasure.Euclidean,
DistanceMeasure.Manhattan,
DistanceMeasure.Maximum,
DistanceMeasure.SquaredEuclidean,
*)
{% endcodeblock %}

{% codeblock lang:fsharp %}
// Compare each subject against every other subject
let comparisons = 
    [0. .. 9.]
    |> List.map (fun x -> 
        [0. .. 9.]
        |> List.filter (fun y -> y <> x)
        |> List.map (fun y -> 
            let d = distance (dataset x 1.0) (dataset y videoIdFilter)
            (x, y, d)))
    |> List.concat
    |> List.sortBy (fun (_, _, d) -> d)

let (subject1, subject2, difference) = 
    comparisons
    |> List.take 1
    |> List.exactlyOne
{% endcodeblock %}

For visualization purposes, I create a comparison chart for each subject against Subject 1.  Then I save the charts to files.

{% codeblock lang:fsharp %}
// Show comparison charts
comparisons
// Only include matches against subject1
|> List.filter (fun (s1, s2, d) -> s1 = subject1)
// Exclude subject1 vs. subject1
|> List.filter (fun (s1, s2, d) -> s2 <> subject1)
// Chart each pair
|> List.iter (fun (s1, s2, d) -> 
    Chart.Combine([
        Chart.FastLine(dataset s1 1.0);
        Chart.FastLine(dataset s2 1.0)])
    |> Chart.WithTitle(String.Format("Subject {0:n0} vs. Subject {1:n0} (distance: {2:n1})", s1, s2, d))
    |> Chart.Save (Path.Combine(__SOURCE_DIRECTORY__, String.Format(@"..\\data\\eeg\\chart_{0:n0}_{1:n0}.png", s1, s2)))
    //|> Chart.Show
    )
{% endcodeblock %}

Here are the top couple matches, in the format (subject1, subject2, distance).

{% codeblock lang:fsharp %}
(float * float * float) list =
[
(1.0, 4.0, 26727187.0); 
(6.0, 0.0, 28741980.0); 
(3.0, 4.0, 29468040.0);
(6.0, 4.0, 30171222.0); 
(8.0, 4.0, 31123338.0); 
(3.0, 0.0, 31511006.0);
(4.0, 0.0, 31697843.0); 
...
]
{% endcodeblock %}

Here is the top signal match.

![Best Match](/images/dtw1/chart_1_4.png)

Here are the other subjects compared to subject 1.

![1 and 0](/images/dtw1/chart_1_0.png)

![1 and 2](/images/dtw1/chart_1_2.png)

![1 and 3](/images/dtw1/chart_1_3.png)

![1 and 5](/images/dtw1/chart_1_5.png)

![1 and 6](/images/dtw1/chart_1_6.png)

![1 and 7](/images/dtw1/chart_1_7.png)

![1 and 8](/images/dtw1/chart_1_8.png)

![1 and 9](/images/dtw1/chart_1_9.png)






