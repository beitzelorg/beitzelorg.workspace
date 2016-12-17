---
title: 'Decision Trees with F# and Accord.NET (Part 1)'
date: 2016-12-05 21:13:22
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

It is time for an exploration into using a Decision Tree to classify Lego set themes using [F#](http://fsharp.org/) and [Accord.NET](http://accord-framework.net/).  

First things first, the data. [Rebrickable](http://rebrickable.com/downloads) has downloadable datasets for Lego sets and pieces.  I'll use the ```sets.csv``` file as the primary dataset driver, but will grab information from ```sets_pieces.csv```, ```pieces.csv```, and ```colors.csv``` for feature creation.  The files are not in an format appropriate for a decision tree, so some transformations will need to happen first.  I don't want the post to get too long, so this project will be broken into two components.  Part 1 will be building the feature file and getting the data into the desired comsumable format, [Part 2](/2016/12/07/Decision-Trees-with-F-and-Accord-NET-Part-2) will actually use the file to get to the end goal.

Second, the approach.  The goal of part 1 is to do all transformations here.  I want the end result to be a file that can be directly loaded into part 2's code.  I will use/create a series of features.  Year is the set's year, and is directly provided.  The following features will need to be grouped and calculated.  First is "% of the set's pieces are <x> type" for a couple major piece types.  Second, is "% of the set's pieces are <x> color" for major color groups.  Lastly, the prediction target is theme.  The dataset has up to three themes per set (T1, T2, T3).  For simplicity sake I am only going to use one theme (T1) as the target theme to predict.  This will restrict the quality of my results, but as a proof-of-concept it will be good enough.  Hopefully all this give me some interesting results.


Using [Paket](https://github.com/fsprojects/Paket), here is a sample ```paket.dependencies``` file.

{% codeblock lang:fsharp %}
source https://nuget.org/api/v2
nuget FSharp.Data
{% endcodeblock %}

This is the boring setup stuff.  

{% codeblock lang:fsharp %}
#r "../packages/FSharp.Data/lib/net40/FSharp.Data.dll"

open System
open System.IO
open FSharp.Data
{% endcodeblock %}

Next it is time to leverage the CsvProvider for the input files.  The below code configures the types as well as loads the data. You've probably read it a million times, but Type Providers are really helpful to get get working with the data quickly.

{% codeblock lang:fsharp %}
// File structures
type LegoSets = CsvProvider<"../data/legos/sets.csv">
type LegoSetPieces = CsvProvider<"../data/legos/set_pieces.csv">
type LegoPieces = CsvProvider<"../data/legos/pieces.csv">
type LegoColors = CsvProvider<"../data/legos/colors.csv">

// Load files
let legoSets = LegoSets.Load "../data/legos/sets.csv"
let legoSetPieces = LegoSetPieces.Load "../data/legos/set_pieces.csv"
let legoPieces = LegoPieces.Load "../data/legos/pieces.csv"
let legoColors = LegoColors.Load "../data/legos/colors.csv"
{% endcodeblock %}

When building the features, I will be counting the number of specific colors in the set.  There are 135 different colors.  I only care about eight different colors: Red, Green, Blue, White, Black, Gray, Silver, and Translucent.  I will ignore the rest.  As an expedient hack, I search for the color text in the description.  So 'Red', 'Trans-Red', and 'Dark Red' all count as 'Red'.  I then store these indexes for later searching.  This method misses things like 'Pink', which is in the red family.  It also means 'Trans-Red' counts as a red piece and a translucent piece.  For a real problem I would be more thorough, but I just want to get to the decision tree.

{% codeblock lang:fsharp %}
let getColorIndexes color =
    legoColors.Rows 
    |> Seq.filter (fun x -> x.Descr.Contains(color)) 
    |> Seq.map (fun x -> x.Id) 
    |> Seq.toList

let redIndexes = getColorIndexes "Red"
let greenIndexes = getColorIndexes "Green"
let blueIndexes = getColorIndexes "Blue"
let whiteIndexes = getColorIndexes "White"
let blackIndexes = getColorIndexes "Black"
let grayIndexes = getColorIndexes "Gray"
let silverIndexes = getColorIndexes "Silver"
let translucentIndexes = getColorIndexes "Trans"
{% endcodeblock %}

To perform piece counts in the sets I'll need to do some grouping.  I will use *SetCountsDetail* as an intermediate aggregation record type.  *SetDetail* will be my final output form.  You may notice I use counts for the aggregation, but in the final output I store "Percent of the set".  I feel this should allow the feature values to be consistent across sets.  I also use the function *setCountsDetailSum* when folding group sums together.

{% codeblock lang:fsharp %}
// Piece-type counts for sets (used for aggregation)
type SetCountsDetail = { 
    SetId: string; 
    BricksCount: int; 
    PlatesCount: int; 
    MinifigsCount: int;
    PanelsCount: int;
    PlantsAndAnimalsCount: int;
    TilesCount: int;
    TechnicsCount: int;
    RedCount: int;
    GreenCount: int;
    BlueCount: int;
    WhiteCount: int;
    BlackCount: int;
    GrayCount: int;
    SilverCount: int;
    TranslucentCount: int}

// Set detail record (this is what gets written to the output file)
type SetDetail = { 
    SetId: string; 
    Year: int;
    ThemeId: int;
    PiecesCount: int; 
    BricksPct: float; 
    PlatesPct: float; 
    MinifigsPct: float 
    PanelsPct: float;
    PlantsAndAnimalsPct: float;
    TilesPct: float;
    TechnicsPct: float;
    RedPct: float;
    GreenPct: float;
    BluePct: float;
    WhitePct: float;
    BlackPct: float;
    GrayPct: float;
    SilverPct: float;
    TranslucentPct: float} with
    static member toCsv d =
        sprintf "%d,%d,%d,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f,%.4f" 
            d.ThemeId d.Year d.PiecesCount d.BricksPct d.PlatesPct d.MinifigsPct d.PanelsPct d.PlantsAndAnimalsPct d.TilesPct d.TechnicsPct d.RedPct d.GreenPct d.BluePct d.WhitePct d.BlackPct d.GrayPct d.SilverPct d.TranslucentPct
    
// Sum counts for SetCountsDetail (use in fold)
let setCountsDetailSum (a:SetCountsDetail) (b:SetCountsDetail) = 
    {a with 
        BricksCount = a.BricksCount + b.BricksCount;
        PlatesCount = a.PlatesCount + b.PlatesCount;
        MinifigsCount = a.MinifigsCount + b.MinifigsCount;
        PanelsCount = a.PanelsCount + b.PanelsCount;
        PlantsAndAnimalsCount = a.PlantsAndAnimalsCount + b.PlantsAndAnimalsCount;
        TilesCount = a.TilesCount + b.TilesCount;
        TechnicsCount = a.TechnicsCount + b.TechnicsCount;
        RedCount = a.RedCount + b.RedCount;
        GreenCount = a.GreenCount + b.GreenCount;
        BlueCount = a.BlueCount + b.BlueCount;
        WhiteCount = a.WhiteCount + b.WhiteCount;
        BlackCount = a.BlackCount + b.BlackCount;
        GrayCount = a.GrayCount + b.GrayCount;
        SilverCount = a.SilverCount + b.SilverCount;
        TranslucentCount = a.TranslucentCount + b.TranslucentCount}

{% endcodeblock %}

Now I create a piece lookup using a Map (for the non-F#ers, think Dictionary).  I also filter only the piece type categories I care about.  There are 56, and for simplicity I will only look at seven.  I also group all "Technic*" categories into a single "Technic" category.

{% codeblock lang:fsharp %}

// Build lookup for pieces
let pieceLookup = 
    legoPieces.Rows
    // Only include these categories
    |> Seq.filter (fun row -> 
        row.Category = "Bricks" || 
        row.Category = "Plates" || 
        row.Category = "Minifigs" ||
        row.Category = "Panels" ||
        row.Category = "Plants and Animals" ||
        row.Category = "Tiles" ||
        row.Category.Contains("Technic")
        )
    |> Seq.map (fun row -> (row.Piece_id, if row.Category.Contains("Technic") then "Technic" else row.Category))
    |> Map

{% endcodeblock %}

Here I take a row and transpose it in the intermediate feature columns I want.  *isColorIndex* is a helper function to determine if the specified piece color is part of one my color groupings.  

{% codeblock lang:fsharp %}
// Convert data row to a SetCountsDetail record
let isColorIndex indexList colorIndex = 
    indexList |> List.filter (fun x -> x = colorIndex) |> List.length > 0

// Convert data row to a SetCountsDetail record
let rowToSetCountsDetail (row:CsvProvider<"../data/legos/set_pieces.csv">.Row) = 
    let c = pieceLookup.Item row.Piece_id
    {        
        SetCountsDetail.SetId = row.Set_id;  
        BricksCount = if c = "Bricks" then row.Num else 0;
        PlatesCount = if c = "Plates" then row.Num else 0;
        MinifigsCount = if c = "Minifigs" then row.Num else 0;
        PanelsCount = if c = "Panels" then row.Num else 0;
        PlantsAndAnimalsCount = if c = "Plants and Animals" then row.Num else 0;
        TilesCount = if c = "Tiles" then row.Num else 0;
        TechnicsCount = if c = "Technic" then row.Num else 0;
        RedCount = if isColorIndex redIndexes row.Color then row.Num else 0;
        GreenCount = if isColorIndex greenIndexes row.Color then row.Num else 0;
        BlueCount = if isColorIndex blueIndexes row.Color then row.Num else 0;
        WhiteCount = if isColorIndex whiteIndexes row.Color then row.Num else 0;
        BlackCount = if isColorIndex blackIndexes row.Color then row.Num else 0;
        GrayCount = if isColorIndex grayIndexes row.Color then row.Num else 0;
        SilverCount = if isColorIndex silverIndexes row.Color then row.Num else 0;
        TranslucentCount = if isColorIndex translucentIndexes row.Color then row.Num else 0}
{% endcodeblock %}


I then create a series of lookup functions to support the transformation process. In *setPiecesLookup* I make a Map for piece counts by SetId.  It gets alittle gnarly, but it does a group by on SetId, then sums all columns up to that level. *getCountLookup* is used to get piece counts by setid. *themesLookup* maps the set's theme text to an arbitrary int.  I will save that into a lookup table/file as well for later access.

{% codeblock lang:fsharp %}
// Build lookup for setpieces
let setPiecesLookup = 
    legoSetPieces.Rows
    |> Seq.filter (fun r -> Map.containsKey r.Piece_id pieceLookup)
    |> Seq.map rowToSetCountsDetail
    // Sum counts up to SetId                             
    |> Seq.groupBy (fun x -> x.SetId)
    |> Seq.map (fun (k, v) -> 
        (k, 
         v |> Seq.fold setCountsDetailSum { SetCountsDetail.SetId = k; BricksCount = 0; PlatesCount = 0; MinifigsCount = 0; PanelsCount = 0;  PlantsAndAnimalsCount = 0; TilesCount = 0; TechnicsCount = 0; RedCount = 0; GreenCount = 0; BlueCount = 0; WhiteCount = 0; BlackCount = 0; GrayCount = 0; SilverCount = 0; TranslucentCount = 0}))                                
    // Create lookup
    |> Map


// Lookup piece count in the set, if not found, return 0
let getCountLookup k =
    match Map.tryFind k setPiecesLookup with
    | Some(x) -> x
    | _       -> { SetId = k; BricksCount = 0; PlatesCount = 0; MinifigsCount = 0; PanelsCount = 0; PlantsAndAnimalsCount = 0; TilesCount = 0; TechnicsCount = 0; RedCount = 0; GreenCount = 0; BlueCount = 0; WhiteCount = 0; BlackCount = 0; GrayCount = 0; SilverCount = 0; TranslucentCount = 0}


// Create theme lookups from the sets files
let themesLookup =
    let distinctThemes = legoSets.Rows |> Seq.map (fun row -> row.T1) |> Seq.distinct

    // Pair theme string with an int
    (distinctThemes, seq [0..Seq.length distinctThemes])
    ||> Seq.zip
    |> Map

{% endcodeblock %}


All the hard work is done.  I now just take the data from the file and run it through a series of filters and transformations to transpose the bricktype and piece color counts into "percent of the set" columns.  Once that is done I write out a file ```aggregatedata.csv``` that will be used in Part 2.  I also save a themes lookup file.  The lookup isn't actually needed for the decision tree processing, but its a nice-to-have if I want to remap the int ids back to text values for evaluation.

{% codeblock lang:fsharp %}
// Build dataset
let dataset = 
    legoSets.Rows
    // Exclude blank themes
    |> Seq.filter (fun row -> not (String.IsNullOrEmpty(row.T1)))
    // Exclude sets with 0 pieces
    |> Seq.filter (fun row -> row.Pieces <> 0)
    |> Seq.map (fun row ->
        // Get item type counts for each row (ie set)
        let bricksCount = (getCountLookup row.Set_id).BricksCount
        let platesCount = (getCountLookup row.Set_id).PlatesCount
        let minifigsCount = (getCountLookup row.Set_id).MinifigsCount
        let panelsCount = (getCountLookup row.Set_id).PanelsCount
        let plantsAndAnimalsCount = (getCountLookup row.Set_id).PlantsAndAnimalsCount
        let tilesCount = (getCountLookup row.Set_id).TilesCount
        let technicsCount = (getCountLookup row.Set_id).TechnicsCount
        let redCount = (getCountLookup row.Set_id).RedCount
        let greenCount = (getCountLookup row.Set_id).GreenCount
        let blueCount = (getCountLookup row.Set_id).BlueCount
        let whiteCount = (getCountLookup row.Set_id).WhiteCount
        let blackCount = (getCountLookup row.Set_id).BlackCount
        let grayCount = (getCountLookup row.Set_id).GrayCount
        let silverCount = (getCountLookup row.Set_id).SilverCount
        let translucentCount = (getCountLookup row.Set_id).TranslucentCount

        // Build a record for writing
        {SetId = row.Set_id;
         Year = row.Year;
         ThemeId = themesLookup.Item row.T1;
         PiecesCount = row.Pieces;
         BricksPct = float(bricksCount) / float(row.Pieces);
         PlatesPct = float(platesCount) / float(row.Pieces);
         MinifigsPct = float(minifigsCount) / float(row.Pieces);          
         PanelsPct = float(panelsCount) / float(row.Pieces); 
         PlantsAndAnimalsPct = float(plantsAndAnimalsCount) / float(row.Pieces); 
         TilesPct = float(tilesCount) / float(row.Pieces);
         TechnicsPct = float(technicsCount) / float(row.Pieces);
         RedPct = float(redCount) / float(row.Pieces);
         GreenPct = float(greenCount) / float(row.Pieces);
         BluePct = float(blueCount) / float(row.Pieces);
         WhitePct = float(whiteCount) / float(row.Pieces);
         BlackPct = float(blackCount) / float(row.Pieces);
         GrayPct = float(grayCount) / float(row.Pieces);
         SilverPct = float(silverCount) / float(row.Pieces);
         TranslucentPct = float(translucentCount) / float(row.Pieces)})


// Write dataset to output file
let dataFile = List.Cons (
    "ThemeId,Year,PiecesCount,BricksPct,PlatesPct,MinifigsPct,PanelsPct,PlantsAndAnimalsPct,TilesPct,TechnicsCount,RedPct,GreenPct,BluePct,WhitePct,BlackPct,GrayPct,SilverPct,TranslucentPct", 
    dataset
    |> Seq.toList
    |> List.map SetDetail.toCsv)
	
File.WriteAllLines(Path.Combine(__SOURCE_DIRECTORY__, @"..\\data\\legos\\aggregatedata.csv"), dataFile)


// Write themes lookup to a file
let themesFile = List.Cons (
    "ThemeId,Theme",
    themesLookup
    |> Map.toList 
    |> List.map (fun (key, value) -> sprintf "%s,%d" key value))

File.WriteAllLines(Path.Combine(__SOURCE_DIRECTORY__, @"..\\data\\legos\\aggregatedata_themes.csv"), themesFile)
{% endcodeblock %}


Here are samples of the aggregate data and lookups files.

File: aggregatedata.csv
{% codeblock lang:csv %}
ThemeId,Year,PiecesCount,BricksPct,PlatesPct,MinifigsPct,PanelsPct,PlantsAndAnimalsPct,TilesPct,TechnicsCount,RedPct,GreenPct,BluePct,WhitePct,BlackPct,GrayPct,SilverPct,TranslucentPct
0,1970,471,0.8110,0.0892,0.0000,0.0000,0.0000,0.0000,0.0000,0.1359,0.0000,0.0212,0.4480,0.0000,0.0828,0.0000,0.0000
1,1978,12,0.0000,0.0000,0.8333,0.0000,0.0000,0.0000,0.0000,0.1667,0.0000,0.1667,0.0000,0.2500,0.0000,0.0000,0.0000
2,1987,2,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000
...
{% endcodeblock %}

File: aggregatedata_themes.csv
{% codeblock lang:csv %}
ThemeId,Theme
4 Juniors,47
Adventurers,28
...
{% endcodeblock %}


[Data Files](/data/legodecisiontree.zip)

So there are the data transformations.  Until next time...

