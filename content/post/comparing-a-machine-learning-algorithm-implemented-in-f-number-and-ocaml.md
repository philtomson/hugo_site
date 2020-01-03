+++
title = "Comparing a Machine Learning algorithm implemented in F Sharp and OCaml"
date = "2014-05-29"
slug = "2014/05/29/2014-05-29-comparing-a-machine-learning-algorithm-implemented-in-f-sharp-and-ocaml"
Categories = []
+++

(UPDATE: please see the [next post: Stop the Presses: OCaml wins in terms of speed](http://philtomson.github.io/blog/2014/05/30/stop-the-presses-ocaml-wins/) in which it is shown that there were some serious flaws in my OCaml implementation below. I'll keep this post around for reference.)

I've been dabbling with OCaml for the last several years and so when I saw a recent Meetup notice about an F# machine learning code dojo being led by [Mathias Brandewinder](https://github.com/mathias-brandewinder) within walking distance of my house I thought I'd check it out. I'm one of those Linux guys who tends to eschew IDEs, prefers to use vim, and considers .NET to be some very foreign territory. F# is based on OCaml, so I figured I'd have a bit of a headstart in the language area even if I haven't done any .NET development.

Also, I was curious as I had been hearing rumors on twitter that F# programs were running on Mono faster than the equivalent OCaml versions.  I was also taking Andrew Ng's Coursera Machine Learning course so the timing seemed good. So I loaded up [mono](http://www.mono-project.com/Main_Page), [monodevelop](http://monodevelop.com/) and F#. As it turns out none of it worked on the day of the dojo, so I looked on with someone with a Windows laptop and Visual Studio.

The code dojo itself was a great way of learning F#. Check out Mathias' [Digits Recognizer code Dojo code and instructions](https://github.com/c4fsharp/Dojo-Digits-Recognizer) which in turn was based on [a Kaggle Machine Learning challenge](http://www.kaggle.com/c/digit-recognizer). 
The task was to implement a [k-NN](http://en.wikipedia.org/wiki/K-nearest_neighbors_algorithm) (K Nearest Neighbor; in this case K=1) algorithm for recognizing handwritten digits from a dataset that was provided: A training set of 5000 labeled examples and a validation set of 500 labeled examples. Each digit example is a set of 28X28 grayscale pixels (784 integer values from 0 to 255). 
We wrote a classifier that reads in the training set and then compares each of the 500 validation samples against those 5000 training exmples and returns the label associated with the closest match (the vector with the shortest distance from a training example). k-NN is one of the simplest machine learning algorithms, even so, we were able to get a 94.4% accuracy rate.

After the Dojo, I spent some time getting Monodevelop and F# running on my Ubuntu 12.04 laptop. Turns out I needed to get Monodevelop 4.0 (built from source) and F# 3.0 (initially tried F# 3.1, but Monodevelop 4.0 wasn't happy with that version). Then I set about reimplementing the algorithm in F# and after I got that working I reimplemented the same algorithm in OCaml. 

Here's my F# implementation:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
module classifyDigits.Main
  
open System
open System.IO

type LabelPixels = { Label: int; Pixels: int[] }
let slurp_file file = 
   File.ReadAllLines(file).[1..] 
   |> Array.map (fun line -> line.Split(','))  
   |> Array.map (fun numline -> Array.map (fun (x:string) -> Convert.ToInt32(x)) numline)    
   |> Array.map (fun line -> { Label= line.[0]; Pixels=line.[1..] })

//load the trainingsample  
let trainingset = slurp_file("/home/phil/devel/f_sharp/Dojo-Digits-Recognizer/Dojo/trainingsample.csv") 
 
//  COMPUTING DISTANCES
 
// We need to compute the distance between images
// Math reminder: the euclidean distance is
// distance [ x1; y1; z1 ] [ x2; y2; z2 ] = 
// sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2) + (z1-z2)*(z1-z2))

let distance (p1: int[]) (p2: int[]) = 
  Math.Sqrt (float(Array.sum (Array.map2 ( fun a b -> (pown (a-b) 2)) p1 p2) ))
 
 
//  WRITING THE CLASSIFIER FUNCTION
 
// We are now ready to write a classifier function!
// The classifier should take a set of pixels
// (an array of ints) as an input, search for the
// closest example in our sample, and predict
// the value of that closest element.
 
let classify (pixels: int[]) =
  fst (trainingset |> Array.map (fun x -> (x.Label, (distance pixels x.Pixels ) ))
                   |> Array.minBy (fun x -> snd x ))
 
// EVALUATING THE MODEL AGAINST VALIDATION DATA
 
// Now that we have a classifier, we need to check
// how good it is. 
// This is where the 2nd file, validationsample.csv,
// comes in handy. 
// For each Example in the 2nd file,
// we know what the true Label is, so we can compare
// that value with what the classifier says.
// You could now check for each 500 example in that file
// whether your classifier returns the correct answer,
// and compute the % correctly predicted.


let _ = 
    Console.WriteLine("start...")
    let validationsample = slurp_file("/home/phil/devel/f_sharp/Dojo-Digits-Recognizer/Dojo/validationsample.csv") 
    let num_correct = (validationsample |> Array.map (fun p -> if (classify p.Pixels ) = p.Label then 1 else 0)
                                        |> Array.sum) 
    Printf.printf "Percentage correct:%f\n" ((float(num_correct)/ (float(Array.length validationsample)))*100.0)

{{< / highlight  >}}

Timing when running this on my Dell XPS 13 (i7 Haswell, 8GB RAM):

    real        0m22.180s
    user        0m22.253s
    sys         0m0.116s

And the OCaml implementation:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
(* OCaml version
   compile with:
    ocamlopt str.cmxa -o classifyDigits classifyDigits.ml 
*)

let read_lines name : string list =
  let ic = open_in name in
  let try_read () =
    try Some (input_line ic) with End_of_file -> None in
  let rec loop acc = match try_read () with
    | Some s -> loop (s :: acc)
    | None -> close_in ic; List.rev acc in
  loop []
  
type labelPixels = { label: int; pixels: int list }
let slurp_file file = 
   List.tl (read_lines file) 
   |> List.map (fun line -> Str.split (Str.regexp ",") line )  
   |> List.map (fun numline -> List.map (fun (x:string) -> int_of_string x) numline)    
   |> List.map (fun line -> { label= (List.hd line); pixels=(List.tl line) })
  
let trainingset = slurp_file("./trainingsample.csv") 

(* 
// COMPUTING DISTANCES
 
// We need to compute the distance between images
// Math reminder: the euclidean distance is
// distance [ x1; y1; z1 ] [ x2; y2; z2 ] = 
// sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2) + (z1-z2)*(z1-z2))
*)

let list_sum lst = List.fold_left (fun x acc -> x+.acc) 0.0 lst

let distance (p1: int list) (p2: int list) = 
  sqrt (list_sum (List.map2 ( fun a b -> (float_of_int(a-b)**2.0)) p1 p2) )
 
(* 
// WRITING THE CLASSIFIER FUNCTION
 
// We are now ready to write a classifier function!
// The classifier should take a set of pixels
// (an array of ints) as an input, search for the
// closest example in our sample, and predict
// the value of that closest element.
*)

let minBy f lst  = 
  let smallest = ref (List.hd lst) in
  List.iter (fun x -> if (f x) < (f !smallest) then smallest := x
                          ) (List.tl lst) ;
  !smallest ;;
                        

let classify (pixels: int list) =
  fst ((List.map (fun (x: labelPixels) -> (x.label, (distance pixels x.pixels) )) trainingset)
       |> minBy (fun x  -> snd x)  )
 
(*
// EVALUATING THE MODEL AGAINST VALIDATION DATA
 
// Now that we have a classifier, we need to check
// how good it is. 
// This is where the 2nd file, validationsample.csv,
// comes in handy. 
// For each Example in the 2nd file,
// we know what the true Label is, so we can compare
// that value with what the classifier says.
// You could now check for each 500 example in that file
// whether your classifier returns the correct answer,
// and compute the % correctly predicted.
*)

let validationsample = slurp_file("./validationsample.csv") 
let num_correct = (validationsample |> List.map (fun p -> if (classify p.pixels ) = p.label then 1. else 0.) |> list_sum) 
let _ = Printf.printf "Percentage correct:%f\n" (((num_correct)/. (float_of_int(List.length validationsample)))*.100.0)
{{< / highlight  >}}

The code is very similar. The OCaml version is a bit longer because I needed to implement some functions that are built into the F# library, specifically, *minBy* (*Array.minBy* is built in to thd F# standard lib), *list_sum* (*Array.sum* is the F# builtin function) and *read_lines* (*File.ReadAllLines* is the F# builtin). One difference that could possibly effect performance: The F# version reads the data into *Array*s whereas the OCaml version reads the data into *List*s. The OCaml version was just easier to read into a *List* and OCaml's *List* module has a *map2* function whereas OCaml's *Array* module does not (F# apparently has a *map2* for both *Array* and *List*). 

(EDIT: But as it turns out, using a List here instead of an Array is probably responsible for about half of the performance difference between the F# and the OCaml version. See [the update]((http://philtomson.github.io/blog/2014/05/30/stop-the-presses-ocaml-wins/) )

I was surprised how much less performant the OCaml version was:

    real	0m47.311s
    user	0m46.881s
    sys		0m0.135s

The F# implementation was slightly more than twice as fast o_O.

Next, I recalled from the dojo that it's very easy to parallelize the maps in F#; instead of *Array.map* you use *Array.parallel.map*. So after a couple of edits to add parallel maps, I rebuilt and ran with the following time:

    real	0m16.400s
    user	0m47.135s
    sys		0m0.240s

So after that very simple change the F# version was very nearly 3X faster than the OCaml version. And in *top* I could see that the CPU usage was up to 360% as it ran.

Ah, but what about OCaml's [Parmap](https://github.com/rdicosmo/parmap)?
I modified my OCaml version to use Parmap... but it took a lot longer to get it working than parallelizing the F# version. Probably about 30 minutes by the time I got it figured out (including figuring out how to compile it).

Here's the OCaml version of the *classify* function and *num_correct* that uses Parmap:
    
{{< highlight ocaml "linenos=table,linenostart=1" >}}
let classify (pixels: int list) =
  let label_dist = Parmap.parmap ~chunksize:50 ~ncores:4 (fun x -> (x.label, (distance pixels x.pixels) )) (Parmap.L trainingset) in
  let min = minBy (fun x y -> compare (snd x) (snd y)) label_dist in
  fst min

let num_correct = 
  (Parmap.L validationsample) |> Parmap.parmap ~ncores:4 (fun p -> if (classify p.pixels ) = p.label then 1. else 0.) |> list_sum 
{{< / highlight >}}
Notice that you have to tell it how many cores you have.

After I got it to compile I ran it with much anticipation...

    real	1m43.140s
    user	5m38.809s
    sys		0m42.811s

... over twice as slow as the original OCaml implementation.
I noticed in *top* that this version launched 4 different *classifyDigitPar* processes so I'm guessing that it uses sockets to communicate between the processes. For this particular problem that doesn't appear to be effective. Perhaps it would be for a longer running example, but I'm not sure. F# certainly wins in the parallelization department.

### Observations

Other than performance (where F# on Mono seems to win handily) I noticed the following:

1. The Array slicing syntax in F# is quite nice. 
2. F#'s standard library is just more comprehensive. I could have probably used Janestreet's *Core* library for OCaml which is quite comprehensive, but I wanted to stick with the standard library for both.
3. Monodevelop has a *vi* editing mode. It's kind of ok. But it's missing a lot of keybindings that you'd have in native vi.
4. I have no idea what F#'s (or Monodevelop's) equavilent to *opam* is. I'm guessing OCaml wins in the package management department. 
5. F#'s polymorphic math operators were nice (no need for *+.*, etc.).

You can find the code above in [my github](https://github.com/philtomson/ClassifyDigits)

### Next Time

I want to implement the algorithm in [Julia](http://julialang.org/). I'm going to guess that Julia's vectorization support means that it will beat both OCaml and F#.

