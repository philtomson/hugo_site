+++
title = "stop the presses ocaml wins"
date = "2014-05-30"
slug = "2014/05/30/stop-the-presses-ocaml-wins"
Categories = []
+++

###Benchmarking is a tricky thing. 

Several sharp eyes noticed some flaws in the OCaml code in my [last post](http://philtomson.github.io/blog/2014/05/29/comparing-a-machine-learning-algorithm-implemented-in-f-number-and-ocaml/).

The main problem was pointed out by Kim Nguyễn in the comments:

> your OCaml version is flawed... First in F# you do every computation in integer and only convert as float at the end (it seems). If you do the same in OCaml, you will gain some speed, that is:
> list_sum should use 0 and +, and distance should use let x = a - b in x*x to compute the square and only call "float" before sqrt, as you do in F# (and therefore also call float at the end on num_correct since list_sum now returns an integer. Also the computation of num_correct should use now 1 and 0 instead of 1.0 and 0.0).

I made the suggested changes to *distance* and *list_sum* :

title = "stop the presses ocaml wins"
let list_sum lst = List.fold_left (fun x acc -> x+acc) 0 lst

let distance (p1: int list) (p2: int list) = 
  sqrt (float_of_int (list_sum (List.map2 ( fun a b -> let diff = a-b in 
                                           diff*diff ) p1 p2) ))

{% endcodeblock %}

Those simple changes yielded a 2X speedup which puts the OCaml implementation on par with the non-parallelized F# version:

    real	0m23.874s
    user	0m23.623s
    sys	        0m0.079s

Then camlspotter pointed out something about *float array* that I'd forgotten:
> In addition, comparing programs using Arrays and Lists is VERY unfair. In this kind of programs with loops you cannot ignore the cost of list construction and deconstruction. In addition, __OCaml's float array has another benefits: its contents are unboxed, while float list does not__.

camlspotter was kind enough to submit an *Array*ized version of the OCaml code:

title = "stop the presses ocaml wins"
(* OCaml version submitted by @camlspotter
   compile with:
    ocamlopt str.cmxa -o classifyDigitsArray classifyDigitsArray.ml 
*)
 
(*
// This F# dojo is directly inspired by the 
// Digit Recognizer competition from Kaggle.com:
// http://www.kaggle.com/c/digit-recognizer
// The datasets below are simply shorter versions of
// the training dataset from Kaggle.
 
// The goal of the dojo will be to
// create a classifier that uses training data
// to recognize hand-written digits, and
// evaluate the quality of our classifier
// by looking at predictions on the validation data.
 
*)
 
let read_lines name : string list =
  let ic = open_in name in
  let try_read () =
    try Some (input_line ic) with End_of_file -> None in
  let rec loop acc = match try_read () with
    | Some s -> loop (s :: acc)
    | None -> close_in ic; List.rev acc in
  loop []
  
(*
 
// Two data files are included in the same place you
// found this script: 
// trainingsample.csv, a file that contains 5,000 examples, and 
// validationsample.csv, a file that contains 500 examples.
// The first file will be used to train your model, and the
// second one to validate the quality of the model.
 
// 1. GETTING SOME DATA
 
// First let's read the contents of "trainingsample.csv"
 
*)
 
type labelPixels = { label: int; pixels: int array }
let slurp_file file = 
  List.tl (read_lines file)
  |> List.map (fun line -> Str.split (Str.regexp ",") line )  
  |> List.map (fun numline -> List.map (fun (x:string) -> int_of_string x) numline)    
  |> List.map (fun line -> 
    { label= List.hd line; 
      pixels= Array.of_list @@ List.tl line })
  |> Array.of_list
  
let trainingset = slurp_file "./trainingsample.csv"
 
(* 
// 6. COMPUTING DISTANCES
 
// We need to compute the distance between images
// Math reminder: the euclidean distance is
// distance [ x1; y1; z1 ] [ x2; y2; z2 ] = 
// sqrt((x1-x2)*(x1-x2) + (y1-y2)*(y1-y2) + (z1-z2)*(z1-z2))
*)
 
let array_fold_left2 f acc a1 a2 =
  let open Array in
  let len = length a1 in
  let rec iter acc i = 
    if i = len then acc
    else 
      let v1 = unsafe_get a1 i in
      let v2 = unsafe_get a2 i in
      iter (f acc v1 v2) (i+1)
  in
  iter acc 0
 
let distance p1 p2 = 
  sqrt 
  @@ float_of_int 
  @@ array_fold_left2 (fun acc a b -> let d = a - b in acc + d * d) 0 p1 p2
 
(* 
// 7. WRITING THE CLASSIFIER FUNCTION
 
// We are now ready to write a classifier function!
// The classifier should take a set of pixels
// (an array of ints) as an input, search for the
// closest example in our sample, and predict
// the value of that closest element.
 
*)
 
let classify (pixels: int array) =
  fst (
    Array.fold_left (fun ((min_label, min_dist) as min) (x : labelPixels) ->
      let dist = distance pixels x.pixels in
      if dist < min_dist then (x.label, dist) else min) 
      (max_int, max_float) (* a tiny hack *)
      trainingset
  )
 
(*
// 8. EVALUATING THE MODEL AGAINST VALIDATION DATA
 
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
 
let validationsample = slurp_file "./validationsample.csv"
 
let num_correct = 
  Array.fold_left (fun sum p -> sum + if classify p.pixels = p.label then 1 else 0) 0 validationsample
 
let _ = 
  Printf.printf "Percentage correct:%f\n" 
  @@ float_of_int num_correct /. float_of_int (Array.length validationsample) *.100.0
{% endcodeblock %}

Now the non-parallelized OCaml version handily beats the parallelized F# version(by about 5 seconds):

    real	0m11.686s
    user	0m11.529s
    sys		0m0.092s

...though the code is a good bit longer than the F# version which weighed in at 58 lines.

###Observations:

1. Be very careful about where you're doing integer vs float math ops; stay with ints as much as possible (so long as you're not sacrificing accuracy).
2. *float array* is preferrable to *float list* because the floats are unboxed in the former. 
3. Don't be obsessed with making an implementation in one language look like the implementation in the comparison language. I think I was focusing too much on the similiarities of the languages to see some of the optimization opportunities. Also, since I wrote the F# version first, I tended to make the OCaml version look like the F# version (usage of *minBy*, for example, and implementing my own *minBy* in OCaml instead of using camlspotter's *Array.fold_left* method.)
4. The parallelization point remains: It's much easier in F#. How fast would this be if OCaml's Parmap used some kind of shared-memory implementation? The F# example was about 1.4X faster with *Parallel* so that would get the OCaml version down to ~8.5 seconds if a similar mechanism existed. 
