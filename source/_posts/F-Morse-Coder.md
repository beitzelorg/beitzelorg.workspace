---
title: 'F# Morse Coder'
date: 2016-12-23 20:49:34
tags:
- F#
- FSharp
- Morse Code
- Sound
- Audio
- Translation
---

## In other words: -.-- .- -.-- / ..-. / ... .... .- .-. .--.

As I was explaining [Morse Code](https://en.wikipedia.org/wiki/Morse_code) to a young mind, I started thinking.  It is fine to explain the encoding and uses, but experiencing the audial component makes the lessons stick better.  Enter [F#](http://fsharp.org/).  Yes, I know I could use any of a hundred phone apps or websites that produce sound, but what's the fun in that?  For me, this is the perfect opportunity to hack out a quick text to morse code translator.  

Getting started, I setup a ```Map``` as the codebook for letter/number to morse code translation.  It's not meant to be comprehensive, but enough to play with.  Then I code in some constants and helpers to make my life easier later. 

{% codeblock lang:fsharp %}
open System
open System.IO

let codebook = 
    Map [ 
        (' ', " "); 
        ('a', ".-");
        ('b', "-...");
        ('c', "-.-.");
        ('d', "-..");
        ('e', ".");
        ('f', "..-.");
        ('g', "--.");
        ('h', "....");
        ('i', "..");
        ('j', ".---");
        ('k', "-.-");
        ('l', ".-..");
        ('m', "--");
        ('n', "-.");
        ('o', "---");
        ('p', ".--.");
        ('q', "--.-");
        ('r', ".-.");
        ('s', "...");
        ('t', "-");
        ('u', "..-");
        ('v', "...-");
        ('w', ".--");
        ('x', "-..-");
        ('y', "-.--");
        ('z', "--..");
        ('1',".----");
        ('2',"..---");
        ('3',"...--");
        ('4',"....-");
        ('5',".....");
        ('6',"-....");
        ('7',"--...");
        ('8',"---..");
        ('9',"----.");
        ('0',"-----")]

let dotDuration = 200
let dashDuration = dotDuration * 3
let letterTrailDuration = dotDuration
let charTrailDuration = dotDuration * 2 // 3
let wordTrailDuration = dotDuration * 6 // 7
let frequency = 700

// Sleep wrapper
let sleep (ms:int) = System.Threading.Thread.Sleep(ms)
{% endcodeblock %}

This is the translation and audio portion.  The approach is basically: string -> char -> code -> dot/dash -> sound. I do lookups in the codebook with ```TryFind```.  This allows me to leverage ```Some``` and ```None```.  For illustrative purposes, the character is displayed as its going audio.  Then the character's code is fed into the morseToSound function.  Here the code is broken apart and the dots (.) and dashes (-) are translated into audio sounds.  Luckily I can just just use ```Console.Beep``` for easy tone creation.  I code spaces as a word seperator and visibilty display unknown characters and patterns with a '!'.  

{% codeblock lang:fsharp %}
// Convert the dot/dash to sound
let dotDashToSound dd = 
    match dd with
    | '.' -> Console.Beep(frequency, dotDuration)
    | '-' -> Console.Beep(frequency, dashDuration)
    | ' ' -> sleep wordTrailDuration
    | _   -> Console.Write("!")
    sleep dotDuration

// Convert morsecode char to sound representation
let morseToSound mc = 
    match mc with 
    | Some(c) ->
        c |> Seq.iter dotDashToSound
        sleep letterTrailDuration
    | None -> Console.Write("!")

// Convert char to sound
let charToSound (c:char) =
    Console.Write("{0}", c)
    morseToSound (codebook.TryFind c)
{% endcodeblock %}

This is a test snippet to make sure it all works.

{% codeblock lang:fsharp %}
let input = "Hello World"
input.ToLower() |> Seq.iter charToSound
{% endcodeblock %}

I want to allow this experience to be more interactive.  It doesn't have to be anything fancy so I just set up a loop that takes a line at a time and does translations.  An empty line exits the program.  

{% codeblock lang:fsharp %}
let rec main() = 
    match Console.ReadLine() with
    | ""    -> ()
    | input -> 
        input.ToLower() |> Seq.iter charToSound
        printfn ""
        main()

main()
{% endcodeblock %}

This mini project is pretty basic.  But it was a quick and fun way to whip up an experimentation tool, and use F# in the process.

