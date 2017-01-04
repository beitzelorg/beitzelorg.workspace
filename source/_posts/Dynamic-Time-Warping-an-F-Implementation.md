---
title: 'Dynamic Time Warping, an F# Implementation'
date: 2016-12-17 22:34:13
tags:
- F#
- FSharp
- DTW
- Dynamic Time Warping
- Signals
- Data
- Similarity
- Optimization
- Tail calls
- Racket
- VS Code
- Ionide
- Mono
---

My recent post about [Dynamic Time Warping](/2016/12/13/F-and-Dynamic-Time-Warping/) used an external library.  It inspired me to implement the algorithm in [F#](http://fsharp.org/).  This is mostly just to see it in F#.  My last implementation was in [Racket](http://racket-lang.org/), and I'm interested in the different language implementations. I use a pretty basic [Algorithm](https://en.wikipedia.org/wiki/Dynamic_time_warping#Implementation), nothing fancy. As part of this process I'll be doing comparisons between [NDtw](https://github.com/doblak/ndtw) and my code.  To be upfront, its not a perfect comparison.  NDtw has additional options and tracking that will reduce it's max performance capabilities.  But for hacking around, the implementations will be close enough for alittle fun.  For anyone interested, unless otherwise specified, all of my results will be from the REPL in [VS Code](https://code.visualstudio.com/) + [Ionide](http://ionide.io/) using [Mono](http://www.mono-project.com/) version 4.6.2.


Using [Paket](https://github.com/fsprojects/Paket), here is a sample paket.dependencies file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget FSharp.Data
nuget NDtw
{% endcodeblock %}

Here is the basic setup stuff.  

{% codeblock lang:fsharp %}
#time
#r "../packages/FSharp.Data/lib/net40/FSharp.Data.dll"
#r "../packages/ndtw/lib/net40/ndtw.dll"

open System
open System.IO
open FSharp.Data
open NDtw
{% endcodeblock %}


First, I need to get some data.  I'm in the mood for some stock data.  That should give me simple signals with lots of data points.  I also decide to pull it into local files.

{% codeblock%}
curl http://ichart.finance.yahoo.com/table.csv?s=IBM -o stock_ibm.csv

curl http://ichart.finance.yahoo.com/table.csv?s=F -o stock_f.csv
{% endcodeblock %}

Here is the code to load the csv data into arrays.  I only pull 5,000 records, and the signal of interest will the "Open" value.  This will be enough data to run long enough, but not too long.

{% codeblock lang:fsharp %}
type Stock = CsvProvider<"../data/stocks/stock_ibm.csv">

let ibm = Stock.Load("../data/stocks/stock_ibm.csv")
let ford = Stock.Load("../data/stocks/stock_f.csv")

let ibmData = ibm.Rows |> Seq.take 5000 |> Seq.map (fun x -> float(x.Open)) |> Seq.toArray
let fordData = ford.Rows |> Seq.take 5000 |> Seq.map (fun x -> float(x.Open)) |> Seq.toArray
{% endcodeblock %}


Before I get started, I want to get a baseline for the NDtw implementation.  I ran this a couple times, and the below results are representative.

{% codeblock lang:fsharp %}
let dtw = new Dtw(ibmData, fordData)
let cost = dtw.GetCost()

//Real: 00:00:01.762, CPU: 00:00:01.781, GC gen0: 128, gen1: 67, gen2: 2
{% endcodeblock %}


Now it is time to implement the algorithm.  This is mostly a copy paste directly off of the wikipedia page.  I just need to do minor F# translation.

{% codeblock lang:fsharp %}
// Distance calculation between 2 points
let distance (a:float) (b:float) = Math.Abs(a - b)

let myDtwArray2D_A (d1:float[]) (d2:float[]) =
    let n = Array.length d1
    let m = Array.length d2
    let path = Array2D.init (n+1) (m+1) (fun _ _ -> 0.)
    [1..n] |> List.iter (fun x -> path.[x,0] <- Double.PositiveInfinity)
    [1..m] |> List.iter (fun x -> path.[0,x] <- Double.PositiveInfinity)

    for i in [1..n] do
        for j in [1..m] do
            let cost = distance d1.[i-1] d2.[j-1]
            path.[i,j] <- cost + (List.min
                [path.[i-1,j];
                path.[i,j-1];
                path.[i-1,j-1]])

    path.[n,m]
{% endcodeblock %}

Here is my first pass.  Again, I ran this multiple times, and this is a representative result. And ouch.  I know functional languages have a reputation for slower performance, but I need to be able to do better.  Not ony is it slower, but the GC #s are 10x worse than my baseline. 

{% codeblock lang:fsharp %}
let cost = myDtwArray2D_A ibmData fordData

// Real: 00:00:10.189, CPU: 00:00:10.234, GC gen0: 1536, gen1: 461, gen2: 1
{% endcodeblock %}

Take 2: When I initially wrote the function I decided to leverage built-ins as much as possible.  As a result I used ```List.min``` when determining which step in the path to take next.  List construction seems like a possible expensive process.  So I'll take that out and see how that does.  The below code is identical to above except for the ```path.[i,j] <- cost + (min path.[i-1,j] path.[i,j-1] path.[i-1,j-1])``` call.  

{% codeblock lang:fsharp %}
// Min of three values
let min (a:float) (b:float) (c:float) =
    if a < b
        then if a < c then a else if b < c then b else c
        else if b < c then b else if a < c then a else c

let myDtwArray2D_B (d1:float[]) (d2:float[]) =
    let n = Array.length d1
    let m = Array.length d2
    let path = Array2D.init (n+1) (m+1) (fun _ _ -> 0.)
    [1..n] |> List.iter (fun x -> path.[x,0] <- Double.PositiveInfinity)
    [1..m] |> List.iter (fun x -> path.[0,x] <- Double.PositiveInfinity)

    for i in [1..n] do
        for j in [1..m] do
            let cost = distance d1.[i-1] d2.[j-1]
            path.[i,j] <- cost + (min
                path.[i-1,j]
                path.[i,j-1]
                path.[i-1,j-1])

    path.[n,m]
{% endcodeblock %}

Well, this is alot better.  The performance is even close to NDtw.  This is decent for a small amount effort.  Taking a closer look, the GC numbers are troubling.  My implementation numbers are high in comparison.  Maybe there is something I can do about that.

{% codeblock lang:fsharp %}
let cost = myDtwArray2D_B ibmData fordData

// Real: 00:00:01.769, CPU: 00:00:01.781, GC gen0: 327, gen1: 57, gen2: 1
{% endcodeblock %}

What can I do?  Well, F# is a functional language with optimized [tail calls](https://en.wikipedia.org/wiki/Tail_recursion).  For those not familar, the TLDR version is to write a recursive call in such a way that the last operation in the function is the return value.  When written in such a way it allows the compiler to effectively unwrap the recursion as a loop with no additional stack frames being allocated.  This is a brief, and unperfect, explanation, so it's worth investigating further.  It is a really powerful construct.  In this particular case my hope is reduced stack allocations means less object creation, so less to garbage collect.  That means it is time to rework the function to be recursive, and more to the point, leverage tail call optimization.

{% codeblock lang:fsharp %}
let myDtwArray2DRecursive (d1:float[]) (d2:float[]) =
    let rec myDtwArray2DRecursive' i j n m (path:float[,]) (d1:float[]) (d2:float[]) =
        if i > n then
            path.[n,m]
        else if j > m then
            myDtwArray2DRecursive' (i + 1) 1 n m path d1 d2
        else
            let cost = distance d1.[i-1] d2.[j-1]
            path.[i,j] <- cost + (min
                path.[i-1,j]
                path.[i,j-1]
                path.[i-1,j-1])

            myDtwArray2DRecursive' (i) (j+1) n m path d1 d2

    let n = Array.length d1
    let m = Array.length d2
    let path = Array2D.init (n+1) (m+1) (fun _ _ -> 0.)
    [1..n] |> List.iter (fun x -> path.[x,0] <- Double.PositiveInfinity)
    [1..m] |> List.iter (fun x -> path.[0,x] <- Double.PositiveInfinity)

    myDtwArray2DRecursive' 1 1 n m path d1 d2
{% endcodeblock %}

This is exciting.  The code is flying now (over 5x faster).  Also note the GC numbers, it appears this last modification worked like a charm.  The code is even running significantly faster than the NDtw code.  To be fair, that isn't an apples to apples comparison, but it is a very encouraging result.  The important take away from this is a minor replace of loops for optimized tail calls can give a pretty satisifying performance boost.

{% codeblock lang:fsharp %}
let cost = myDtwArray2DRecursion ibmData fordData

// Real: 00:00:00.367, CPU: 00:00:00.375, GC gen0: 0, gen1: 0, gen2: 0
{% endcodeblock %}

I like to cut numbers a couple different ways.  Now I build a mini framework to run each algorithm and report its Stopwatch time.  I store these in a list and then report on average performance.  Just to make sure my datasets and calls don't accidently get memoized or cached, I modify the datasets on each iteration run.  It probably isn't necessary, but its an easy way to protected against black magics trying to help me when I don't want the performance help.

{% codeblock lang:fsharp %}
// Test algorithm, time with stopwatch, add to performance results
let testAlgorithm description fn perfResults =
    let stopWatch = System.Diagnostics.Stopwatch.StartNew()
    let cost = fn()
    stopWatch.Stop()
    printfn "%20s = %f Time: %f" description cost stopWatch.Elapsed.TotalMilliseconds
    List.Cons((description, stopWatch.Elapsed.TotalMilliseconds), perfResults)

// Run multiple trials
let mutable perfResults:(string * float) list = []
let rand = new Random()
[0..10]
|> List.iter (fun _ ->    
    let ibmData' = ibmData |> Array.map (fun x -> x + rand.NextDouble() * 5.)
    let fordData' = fordData |> Array.map (fun x -> x + rand.NextDouble() * 5.)

    perfResults <- testAlgorithm "NDtw" (fun () ->
        let dtw = new Dtw(ibmData', fordData')
        dtw.GetCost()) perfResults

    perfResults <- testAlgorithm "Array2D_A" (fun () ->
        myDtwArray2D_A ibmData' fordData') perfResults

    perfResults <- testAlgorithm "Array2D_B" (fun () ->
        myDtwArray2D_B ibmData' fordData') perfResults

    perfResults <- testAlgorithm "Array2DRecursive" (fun () ->
        myDtwArray2DRecursive ibmData' fordData') perfResults
    )

// Show aggregate results
printfn "%20s %8s %8s %8s" "algo" "avg" "min" "max"
perfResults
|> List.groupBy fst
|> List.map (fun (k,v) -> 
    (k, 
     List.averageBy snd v, 
     (List.min (List.map snd v)),
     (List.max (List.map snd v))))
|> List.sortBy (fun (_, avg, _, _) -> avg)
|> List.iter (fun (k, avg, min, max) -> 
    printfn "%20s %8.1f %8.1f %8.1f" k avg min max)
{% endcodeblock %}

Here are the average/min/max runtimes for each function.  I like to keep an eye on differences in run environments, so I run a couple different iterations using the REPL, fsharpi (Mono), and fsi (.NET CLR).  The results are below.

{% codeblock lang:fsharp %}
// REPL
                algo      avg      min      max
    Array2DRecursive    353.2    329.1    498.8 // TCO!
           Array2D_B   1750.2   1681.5   1966.7 // Using custom min function
                NDtw   1975.5   1713.1   2322.1 // Baseline
           Array2D_A  11874.8  11627.4  12141.0 // Using List.min
{% endcodeblock %}

{% codeblock lang:fsharp %}
// fsharpi dtwcompare.fsx
                algo      avg      min      max        
    Array2DRecursive    547.5    507.0    592.4        
                NDtw   1055.8    999.8   1317.1        
           Array2D_B   1552.8   1522.3   1633.7        
           Array2D_A   6581.2   4409.3   8533.6        
{% endcodeblock %}

{% codeblock lang:fsharp %}
// fsi dtwcompare.fsx (fsi version 4.40.23020.0)
                algo      avg      min      max
    Array2DRecursive    659.4    556.6    757.3
                NDtw   1964.3   1819.1   2097.0
           Array2D_B   2903.6   2821.2   3046.9
           Array2D_A  13808.5  13061.0  14632.9
{% endcodeblock %}

The comparative performance is similar, although NDtw and version B flip flop positions.  Those two seem to run about the same, so that probably has more to do with what's going on my machine at the time.  I did expect faster performance with fsi than fsharpi, so that is a bit of surprise.  It is just a reminder that assumptions should always be tested.  Investigating that further may be worthy of a blog post itself.  This has been an interesting examination into implementing a DTW algorithm.  It turned into more of an optimization exercise than I expected, which was a pleasant turn of events.  I hope this has been useful, and inspired more F# algorithm implementations, more dynamic time warping, and more tail calls! 




