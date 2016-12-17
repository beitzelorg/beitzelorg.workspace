---
title: 'Decision Trees with F# and Accord.NET (Part 2)'
date: 2016-12-07 22:12:19
tags:
- F#
- FSharp
- Accord.NET
- Decision Trees
- Machine Learning
- Legos
- Data
- Classification
---

This is part 2 of my attempt to use an [Accord.NET](http://accord-framework.net/) Decision Tree to classify Lego set themes using [F#](http://fsharp.org/).  Just as a reminder, the original data source is [Rebrickable](http://rebrickable.com/downloads).  You can find [Part 1](/2016/12/05/Decision-Trees-with-F-and-Accord-NET-Part-1) here.

Using [Paket](https://github.com/fsprojects/Paket), here is a sample paket.dependencies file.


{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget Accord
nuget Accord.Math
nuget Accord.Statistics
nuget Accord.MachineLearning
nuget FSharp.Data
{% endcodeblock %}

This is the boring setup stuff.  It also includes some utility functions.  Not specific just to the utility functions, but one of my goals for these functions is flexibility.  I can add and remove features from the extract script without impacting code here.  This dynamic aspect of function building makes testing changes easier.

{% codeblock lang:fsharp %}
#i "../packages"
#r "../packages/Accord/lib/net45/accord.dll"
#r "../packages/Accord.MachineLearning/lib/net45/accord.machinelearning.dll"
#r "../packages/Accord.Math/lib/net45/Accord.Math.dll"
#r "../packages/Accord.Math/lib/net45/Accord.Math.Core.dll"
#r "../packages/Accord.Statistics/lib/net45/Accord.Statistics.dll"
#r "../packages/FSharp.Data/lib/net40/FSharp.Data.dll"

open System
open System.IO
open Accord
open Accord.MachineLearning
open Accord.MachineLearning.DecisionTrees
open Accord.MachineLearning.DecisionTrees.Learning
open Accord.Math
open Accord.Statistics.Analysis
open FSharp.Data

let rand = new Random()

// Shuffle an array (in-place)
let shuffle (a:'a[]) =
    let swapByIndex i j =
        let tmp = a.[i]
        a.[i] <- a.[j]
        a.[j] <- tmp
    
    let maxIndex = Array.length a - 1
    [|0..maxIndex|] 
    |> Array.iter (fun i -> swapByIndex i (rand.Next maxIndex))
    a

// Send a tree + input + output and generate a tuple with results for comparison
let getResults (tree:DecisionTree) (inputs:float[][]) (outputs:int[]) =
    Array.zip inputs outputs
    |> Array.map (fun (i,o) -> (i, o, tree.Decide(i)))


// Calculate number of correct predictions
let resultsTotalCorrect results = 
    results
    |> Array.map (fun (_, actual:int, predicted:int) -> 
        if actual = predicted then 1 else 0)
    |> Array.fold (+) 0


// Use the input and output datasets to get correct prediction stats
let processResults tree inputs outputs = 
    let results = getResults tree inputs outputs
    (
        results, 
        resultsTotalCorrect results
    )

// Display results
let showResults description correct total =
    printfn "%s - Direct row match  : Correct %d/%d (%.3f)" description correct total (float(correct) / (float(total)))
{% endcodeblock %}

Here are some data transformation functions. *buildTrainTestIndexes* generates a list of indexes for the training and test sets.  The data is randomized and all records are in one and only one set (no overlap between train and test).  *splitDataset* does the actual split into train and test sets.  *splitDataIntoInputAndOutput* breaks a dataset into inputs and outputs for decision tree consumption.

{% codeblock lang:fsharp %}
// Build a list of indexes for the train and test sets
// Current implementation assigns a random 'trainpercent' of the indexes
// to the trainingset, and the remainder to the test set
let buildTrainTestIndexes (length:int) (trainPercent:float) :(int[] * int[]) = 
    let splitIndex = int(Math.Floor(trainPercent * float(length-1)))
    let indexes = [|0..length - 1|]
    shuffle indexes |> ignore
    (indexes.[0..splitIndex],  // training indexes
     indexes.[splitIndex+1..])  // testing indexes


// Split a dataset into a trainingset and a testing set (with no overlap)
let splitDataset (d:list<'a>) (trainPercent:float) =
    let (trainIndexes, testIndexes) = buildTrainTestIndexes (List.length d) trainPercent
    (trainIndexes |> Array.map (fun i -> List.item i d),  // training set
     testIndexes  |> Array.map (fun i -> List.item i d))  // testing set


// Splits dataset into its input and output 
// Assumption: row's first column is label (output)
let splitDataIntoInputAndOutput (d:'a[][]) = 
    (
        d
        |> Array.map (fun row -> row.[1..(Array.length row - 1)]),
        d
        |> Array.map (fun row -> row.[0])
    )
{% endcodeblock %}

Here are decision tree setup specific functions.  The decision tree uses a DecisionVariable collection. Decision Tree ranges come in different flavors, but all my features are double, thus DoubleRange.  The other point of interest is *decisionVariablesIList*, this is necessarily because the F# list as I was using it didn't meet the interface needs.  This very well could be something I missed on my part, but this seemed like the only way to resolve the conflict.

{% codeblock lang:fsharp %}
let newRangeFromColumn (d:Double[][]) (i:int) = 
    new DoubleRange(
        d |> Array.map (fun r -> r.[i]) |> Array.min,
        d |> Array.map (fun r -> r.[i]) |> Array.max)

// Create a new decision variable for a column
let newDecisionVariableFromColumn (d:Double[][]) (i:int) =
    new DecisionVariable((sprintf "col%d" i), newRangeFromColumn d i)

// Create a list of decision variables for the DecisionTree
let decisionVariables (inputs:float[][]) = 
    [0..(Array.length inputs.[0])-1] 
    |> List.map (newDecisionVariableFromColumn inputs)

// Create a list of decision variables for the DecisionTree
let decisionVariablesIList (inputs:float[][]) = 
    let variableList = new Collections.Generic.List<DecisionVariable>()
    decisionVariables inputs |> List.iter (fun x -> variableList.Add(x))
    variableList

// Get number of classes 
let numClasses (d:int[]) = Array.max d - Array.min d + 1

{% endcodeblock %}

All the prep functions are in place.  First I load the data.  Often I use the CsvProvider, but in this case I want the data directly in an array.

{% codeblock lang:fsharp %}
let dataset = 
    Path.Combine(__SOURCE_DIRECTORY__, "..\\data\\legos\\aggregatedata.csv")
    |> File.ReadLines
    |> Seq.skip 1 // Skip header row
    //|> Seq.take 10000 // For testing only take a subset of records
    |> Seq.map (fun x ->x.Split ',')
    |> Seq.map (fun x -> Array.map (fun y -> float(y)) x)
    |> Seq.toList
{% endcodeblock %}

Here I split data into train and test sets, where 70% of the data is train.

{% codeblock lang:fsharp %}
// Split dataset into train and test sets 
let (trainData, testData) = splitDataset dataset 0.7
printfn "Dataset Sizes: All: %d Train: %d Test: %d" (List.length dataset) (Array.length trainData) (Array.length testData)
{% endcodeblock %}

Now I split the train and test sets into input/output arrays.

{% codeblock lang:fsharp %}
// Split training sets into seperate components for decision tree
let (trainInputs, trainOutputs) = splitDataIntoInputAndOutput trainData    
let (testInputs, testOutputs) = splitDataIntoInputAndOutput testData    
{% endcodeblock %}

Here I create and train the tree.  I use the C4.5 algorithm for the learning method.  Accord also offers ID3 for learning as well.

{% codeblock lang:fsharp %}
// Build the decision tree from training data
let tree = new DecisionTree(decisionVariablesIList trainInputs, numClasses (Array.map int trainOutputs))

// Train the tree
let c45 = new C45Learning(tree)

let error = c45.Learn(trainInputs, (Array.map int trainOutputs))
{% endcodeblock %}

Once the tree is trained, I apply the results to the train and test sets and then display the results.

{% codeblock lang:fsharp %}
// Process train and test results
let (trainResults, trainTotalCorrect) = processResults tree trainInputs (Array.map int trainOutputs)    
let (testResults, testTotalCorrect) = processResults tree testInputs (Array.map int testOutputs)

// Final display of results
showResults "Train" trainTotalCorrect (Array.length trainInputs)
showResults "Test " testTotalCorrect (Array.length testInputs)
{% endcodeblock %}

Below are the results.  They are disappointing, and there is certainly room for improvement.  But its a start. 

```
Train - Direct row match  : Correct 5624/7584 (0.742)
Test  - Direct row match  : Correct 1342/3250 (0.413)
```

On a whim I attempt to use the ID3 learner instead.  This method requires discrete values.  It's easy enough to convert all the doubles to ints.  Knowing that most of my columns are percentages, I multiple by 100, then convert to an int.  Unfortunantly, my system ran out of memory on this test.  I used fsharpi (Mono) as well as fsi (.NET CLR), but it gave me the same issue.  This deserves some additional follow-up, but I don't have time for that rabbit-hole right now.  Below is the code I used to try the ID3 learner, if someone sees what is wrong, feel free to drop me a line.

{% codeblock lang:fsharp %}

// Below is the hacked up ID3 variant of my test

let newRangeFromColumnInt (d:int[][]) (i:int) = 
    new IntRange(
        d |> Array.map (fun r -> r.[i]) |> Array.min,
        d |> Array.map (fun r -> r.[i]) |> Array.max)

let newDecisionVariableFromColumnInt (d:int[][]) (i:int) =
    new DecisionVariable((sprintf "col%d" i), newRangeFromColumnInt d i)

let decisionVariablesInt (inputs:int[][]) = 
    [0..(Array.length inputs.[0])-1] 
    |> List.map (newDecisionVariableFromColumnInt inputs)

let decisionVariablesIListInt (inputs:int[][]) = 
    let variableList = new Collections.Generic.List<DecisionVariable>()
    decisionVariablesInt inputs |> List.iter (fun x -> variableList.Add(x))
    variableList

let inputsInt i = 
    (Array.map (fun x -> Array.map (fun y -> int(y * 100.)) x) i)
    
let treeId3 = new DecisionTree(decisionVariablesIListInt (inputsInt trainInputs), numClasses (Array.map int trainOutputs))

let id3 = new ID3Learning(treeId3)

id3.ParallelOptions.MaxDegreeOfParallelism <- 1
let errorId3 = id3.Learn((inputsInt trainInputs), (Array.map int trainOutputs))

let (trainResultsId3, trainTotalCorrectId3) = processResultsId3 treeId3 (inputsInt trainInputs) (Array.map int trainOutputs)    
let (testResultsId3, testTotalCorrectId3) = processResultsId3 treeId3 (inputsInt testInputs) (Array.map int testOutputs)

showResults "Train" trainTotalCorrectId3 (Array.length trainInputs)
showResults "Test " testTotalCorrectId3 (Array.length testInputs)
{% endcodeblock %}


Although the end results were anticlimatic, it's nice to see it all come together.  One consolation is with over 100 possible themes, 41% on the test set isn't the worst thing in the world.  Hopefully this has offered some insight into how to use a decision tree in Accord.NET.  Until next time...




