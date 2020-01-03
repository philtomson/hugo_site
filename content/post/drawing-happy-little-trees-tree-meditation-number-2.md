+++
title = "Drawing Happy Little Trees: tree meditation number 2"
date = "2014-05-14"
slug = "2014/05/14/drawing-happy-little-trees-tree-meditation-number-2"
Categories = []
+++

![Alt Ash Tree](http://upload.wikimedia.org/wikipedia/commons/e/eb/Ash_Tree_-_geograph.org.uk_-_590710.jpg )


*In the [previous Tree Meditation](http://philtomson.github.io/blog/2014/04/29/coding-pretty-little-trees-tree-meditation-number-1/) we pondered some preliminary ideas about treeness. Now let's think about how to draw our trees. Before we start, make sure you paste this tree type code from the last installment into the [Try OCaml REPL](http://try.ocamlpro.com/):*

{{< highlight ocaml "linenos=table,linenostart=1" >}}
type 'a tree = Empty | Leaf of 'a | Node of 'a tree * 'a * 'a tree
{{< / highlight >}}

Last time we coded our little tree using the constructors *Empty*, *Leaf*, and *Node* defined in our tree type:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
Node(Leaf 2, 1, Leaf 3) ;;
- : int tree = Node (Leaf 2, 1, Leaf 3)
{{< / highlight >}}

*NOTE: if you're using your own local OCaml REPL (ocaml or utop) make sure you terminate all of the entries here with double semicolon: ;;  The Try Ocaml REPL seems to do this for you automatically.*

To create a picture of the tree we'll use [Graphviz](http://www.graphviz.org/) which is a suite of commandline tools that can be used to visualize graphs. Since our tree is a [graph](http://en.wikipedia.org/wiki/Tree_\(graph_theory\)), we can use Graphviz to visualize it. Specifically, we'll use a program called *dot* to create a visual representation of the graph.

This is what our tree should look like in the *dot* language:


    graph tree {
      {N1[label="1"]}-- {L2[shape=box,label="2"]}
      {N1[label="1"]}-- {L3[shape=box,label="3"]}
    }
 

We have a graph called *tree*. Inside of the *tree* block we define the connections between the nodes (or in our case between *Node*s and/or *Leaf*s, but no *Empty*s, of course, as they're invisible).

Each node in the graph needs a unique identifier which is what *N1,L2* and *L3* are above. We also give each node a label which is what is shown
in the graphical output of the tree. That's the *[label="some label"]* part of the node specifications above.

The '--' means that there is a connection between two nodes.

It's a fairly simple representation of our tree. We can generate the picture of the tree by running: 

    $ dot -Tsvg -o happy_tree.svg tree.dot

This will generate the file happy_tree.svg which we see here:

![Alt Our Happy Tree]( /img/happy_tree.svg)

That's nice and all, but what if our tree is much larger than this one and has many nodes and leaves? We'd like to be able to generate the *tree.dot* file automatically from our coded representation of our tree. To do that we need to talk a bit about tree *traversal* (visiting each node or leaf of our tree structure). 

Let's create a somewhat bigger tree to make our traversal discussion clearer:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let t = Node (
              Node (
                    Leaf "0", 
                    "1", 
                    Leaf "2"), 
              "3", 
              Node (
                    Leaf "4", 
                    "5", 
                    Leaf "6"
              ) 
         )
{{< / highlight >}}

Which looks like: 


![Alt Our Happy Tree]( /img/bigger_tree.svg)

So why the odd order of the numbers in the tree above? One thing that's not clear in our type definition from the previous article is that, by convention, an important property of binary trees is that child trees to the left of the current node should contain values that are less than the value held in the current node and child trees to the right of the current node should contain values that are greater than the value of the current node.

(As an aside, if we wanted to encode that requirement in our type definition we would need to be using a language which has [*dependent types*](http://en.wikipedia.org/wiki/Dependent_type) )

Our chosen ordering above will become clearer as we talk about different types of traversal.

## Tree Traversal

In order to generate a dot file that represents our tree we'll need to *traverse* the nodes of our tree meaning we need to somehow visit all of the nodes of our tree structure.

There are two categories of tree traversal: *Depth First* or *Breadth First (aka Level Order Traversal)*. 

In all cases we start the tree traversal from the root of the tree. In our tree above, the root of the tree is the node that contains the value *3*. *(For whatever reason, in Computer Science trees are generally upside-down with the root at the top and the leaves at the bottom.)*

### Depth First Traversal

There are three ways to do a depth-first traversal of a tree:

#### 1. Pre-order Traversal

The steps of a Pre-order traversal:

1. Visit the current node and do something with the value found there.
2. Traverse the left subtree.
3. Traverse the right subtree.

An OCaml function to do a preorder traversal is defined as follows:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let rec preorder_traverse t = match t with
  | Empty           -> ()
  | Leaf  value     -> Printf.printf "%s " value
  | Node (l,value,r)-> Printf.printf "%s " value; 
                       preorder_traverse l; 
                       preorder_traverse r  

{{< / highlight >}}

The recursive *preorder_traverse* function (note the *rec* in the function definition) takes a tree and uses [pattern matching](http://en.wikipedia.org/wiki/Pattern_matching) to determine what to do with each variant in the *tree* type. If *t* is an *Empty* we do nothing (empty parens *()* also known as *unit* in OCaml) . If It's a *Leaf* we print it's value. If it's a *Node* we print it's value and then call *preorder_traverse* on the left tree of the node and then call *preorder_traverse* on the right tree of the node.

Running this on our tree above we get (You've been typing the code into the [online OCaml REPL](http://try.ocamlpro.com/) right?) :

    # preorder_traverse t ;;
    3 1 0 2 5 4 6 - : unit = ()

#### 2. In-order Traversal

The steps of an In-order traversal:

1. Traverse the left subtree.
2. Visit current node and do something with value found there.
3. Traverse the right subtree.

Here's the OCaml function for an inorder traversal:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let rec inorder_traverse t = match t with
  |  Empty            -> ()
  |  Leaf value       -> Printf.printf "%s " value
  |  Node (l,value,r) -> inorder_traverse l ;
                         Printf.printf  "%s " value; 
                         inorder_traverse r 

{{< / highlight >}}

Running this on our tree we get:

    # inorder_traverse t ;;
    0 1 2 3 4 5 6 - : unit = ()

Now you can get an idea of why we arranged the values in our tree as we did. 

#### 3. Post-order Traversal

Now you probably get the idea. For postorder we're going to traverse the left subtree then traverse the right subtree and finally visit the current node and do something with it's value:


{{< highlight ocaml "linenos=table,linenostart=1" >}}
let rec postorder_traverse t = match t with
  |  Empty  -> () 
  |  Leaf value -> Printf.printf "%s " value 
  |  Node (l,value,r) ->  postorder_traverse l; 
                          postorder_traverse r; 
                          Printf.printf  "%s " value 
{{< / highlight >}}

And when we run this on our tree we get:

    # postorder_traverse t ;;
    0 2 1 4 6 5 3 - : unit = ()

*As an aside, instead of just printing the current value we'd probably want to make each of our traversal functions more general. This is functional programming, after all, and one of the great "wins" of functional programming is being able to pass functions to functions - function composition. We can generalize our traversal functions by passing in a function that will do something with the value:*


{{< highlight ocaml "linenos=table,linenostart=1" >}}
let rec postorder_traverse f t = match t with
  |  Empty            -> ()
  |  Leaf value       -> f value
  |  Node (l,value,r) -> postorder_traverse f l; 
                         postorder_traverse f r; 
                         f value 
{{< / highlight >}}

*Then we could use this version of postorder_traverse to print the values in our tree:*

    # postorder_traverse (fun n -> Printf.printf "%s " n) t;;
    0 2 1 4 6 5 3 - : unit = ()

    

Notice that none of these depth first traversals of the tree will work for us in creating the *dot* file for graphviz. We need some other way to traverse the tree...

We need something that gives us (not actual *dot* code, but you get the idea):
    3 -> 1
    3 -> 5
    1 -> 0
    1 -> 2
    5 -> 4
    5 -> 6

Which brings us to...

### Breadth First (or Level Order) Traversal

A Level Order traversal means we're traversing each level of the tree. A Level order traversal of our tree above would look like(line breaks added to emphasize the levels of the tree):

    3 
    1 5 
    0 2 4 6

This still isn't quite what we need yet, but it seems we're getting closer.

Before we go on, let's define a couple of helper functions that we'll need to construct our *dot* file. 

First off, we'll need to construct a string that gets written to the *dot* file. It would be good to have a function that converts each of the 3 variants that can make up a *tree* type to a string in *dot* format that represents the node in the graph:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let to_dot_node (t: 'a tree) : string = match t with 
  | Empty -> "" 
  | Node(_,x,_) -> "{N"^x^"[label=\"" ^x^"\"]}"
  | Leaf x ->      "{L"^x^"[shape=box,label=\"" ^x^ "\"]}" 
{{< / highlight >}}

Notice that in this function definition we explicitly specified that the *t* argument passed to this function should be of type *'a tree* and the output is of type *string* - this is completely optional as the compiler can figure out the types of the arguments using *type inference*, but it can make the code easier to read. *(Note: ^ is the string concatenation operator in OCaml)*

If you pasted that function in the REPL you'll see:

    val to_dot_node : string tree -> string = <fun> 

Which means that this is a function that takes a *string tree* and returns a *string*.

We can try running this function on our tree:

    # to_dot_node t ;;
    - : string = "{N3[label=\"3\"]}"

Which is the label of the root of our tree (3).

Next, we'll define a function that given two inputs of type *'a tree* will return an *edge* (a line between two nodes in the tree) in *dot* format:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let edge (n1: 'a tree ) (n2 : 'a tree ) : string = (to_dot_node n1)^"--"^(to_dot_node n2)^"\n" 
{{< / highlight >}}

Now we can go on to define our *tree_to_dot* function:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let rec tree_to_dot  acc t = 
  match t with 
  | Empty -> acc
  | Leaf _ as leaf ->   acc ^ (to_dot_node leaf)
  | Node( (Node (_,_,_) as n) , _,  (Leaf _ as leaf)) 
  | Node( Leaf _ as leaf, _, (Node (_,_,_) as n))     ->
      (edge t n) ^ (edge t leaf) ^(tree_to_dot acc n)
  | Node( ( Node(_,_,_)  as n) , _,  Empty)
  | Node( Empty, _, (Node(_,_,_) as n))    ->
      (edge t n) ^ (tree_to_dot acc n)
  | Node( ( Leaf _  as leaf), _, Empty) 
  | Node( Empty, _,(Leaf _ as leaf))   -> 
      (edge t leaf) 
  | Node( (Leaf _ as left), _, (Leaf _ as right)) ->
      (edge t left) ^ (edge t right)
  | Node( Empty, _, Empty) ->
      acc
  | Node( (Node (_,_,_) as nl) , _, (Node(_,_,_) as nr)) -> 
      (edge t nl)^(edge t nr)^(tree_to_dot acc nl)^(tree_to_dot acc nr)
{{< / highlight >}}

The first argument to *tree_to_dot* is *acc* which is specified as being a string. *acc* is our accumulator: we'll use it to build up our *dot* file. Notice that the function is specified as being recursive ( the *rec* specifies that the function can call itself).

Our pattern match covers all of the potential cases we might encounter. The first one matches *t* with *Empty* and in that case returns our accumulator *acc*. The next match is against *Leaf _* (underscore there meaning that the *Leaf* could contain any value). If *t* is a Leaf, then we'll append the *Leaf*'s *dot* node format to the accumulator. 

After this we do several matches where *t* is a *Node* variant. In order to create an *edge* we need to know about the children of the current *Node*. Pattern Matching allows us to do this, the first pattern match against *Node* is: 

      | Node( (Node (_,_,_) as n) , _,  (Leaf _ as leaf))
      | Node( Leaf _ as leaf, _, (Node (_,_,_) as n))     ->
          (edge t n) ^ (edge t leaf) ^(tree_to_dot acc n)

We're considering two cases here: 

  1. The case where the *Node* contains a *Node* as the left sub-tree and a *Leaf* in the right sub-tree.
  2. The case where the *Node* contains a *Leaf* as the left sub-tree and a *Node* in the right sub-tree.

If either of these cases match, we then create an edge from the current *Node* (which is *t*) to the next *Node* (which is *n*) and an edge from the current *Node* (*t*) to the *Leaf* (*leaf*). And we concatenate that with the recursive call to *tree_to_dot acc n*, meaning we're passing the child node *n* on to *tree_to_dot* for further processing. 

Next we match the two cases where there's a single child *Node* and a single *Empty* node. If so we create an edge from the current *Node* (*t*) to the child *Node* and concatenat that with the call to *tree_to_dot* given the child *Node* *n*.

After that we match the two cases where one child of the *Node* is a *Leaf* and the other is *Empty*. If these cases match we only return an edge from the current *Node* *t* to the child *Leaf*. No recursive call to *tree_to_dot* in this case. Since there are no child *Node*s, there's no reason to.

If both sub-trees of the current *Node* *t* are *Leaf*s then the next pattern matches and we concatenate the two edges from the current *Node* *t* to each *Leaf*.

In the next case we check for both sub-trees of the *Node* being *Empty* and if that case matches we just return the accumulator.

Finally, we cover the case where both sub-trees of the current node are *Node*s. Notice that if this case matches we have two calls to *tree_to_dot* - one for the left subtree and the other for the right subtree.


Now we just need to do a bit of housekeeping to write out the complete *dot* file:

{{< highlight ocaml "linenos=table,linenostart=1" >}}
let tree_to_dotfile t file = 
  let dot_tree = "graph tree {\n"^(tree_to_dot " " t)^"}" in
  let channel = open_out file in
  output_string channel dot_tree;
  close_out channel
{{< / highlight >}}

This function won't work in the Try OCaml REPL since it writes to a file. But maybe by now you've been convinced to install OCaml. If so, here's the whole program so you can compile it:


{{< highlight ocaml "linenos=table,linenostart=1" >}}
type 'a tree = Empty | Leaf of 'a | Node of 'a tree * 'a * 'a tree

let to_dot_node (t: 'a tree) : string = match t with 
  | Empty -> "" 
  | Node(_,x,_) -> "{N"^x^"[label=\"" ^x^"\"]}"
  | Leaf x ->      "{L"^x^"[shape=box,label=\"" ^x^ "\"]}" 
 
let edge (n1: 'a tree ) (n2 : 'a tree ) : string = (to_dot_node n1)^"--"^(to_dot_node n2)^"\n"

let rec tree_to_dot (acc : string) (t : 'a tree) : string = 
  match t with 
  | Empty -> acc
  | Leaf _ as leaf ->   acc ^ (to_dot_node leaf)
  | Node( (Node (_,_,_) as n) , _,  (Leaf _ as leaf)) 
  | Node( Leaf _ as leaf, _, (Node (_,_,_) as n))     ->
      (edge t n) ^ (edge t leaf) ^(tree_to_dot acc n)
  | Node( ( Node(_,_,_)  as n) , _,  Empty)
  | Node( Empty, _, (Node(_,_,_) as n))    ->
      (edge t n) ^ (tree_to_dot acc n)
  | Node( ( Leaf _  as leaf), _, Empty) 
  | Node( Empty, _,(Leaf _ as leaf))   -> 
      (edge t leaf) 
  | Node( (Leaf _ as left), _, (Leaf _ as right)) ->
      (edge t left) ^ (edge t right)
  | Node( Empty, _, Empty) ->
      acc
  | Node( (Node (_,_,_) as nl) , _, (Node(_,_,_) as nr)) -> 
      (edge t nl)^(edge t nr)^(tree_to_dot acc nl)^(tree_to_dot acc nr)

let tree_to_dotfile t file = 
  let dot_tree = "graph tree {\n"^(tree_to_dot " " t)^"}" in
  let channel = open_out file in
  output_string channel dot_tree;
  close_out channel;;

let t = Node (
              Node (
                    Leaf "0", 
                    "1", 
                    Leaf "2"), 
              "3", 
              Node (
                    Leaf "4", 
                    "5", 
                    Leaf "6"
              ) 
         )



let _ = tree_to_dotfile t "tree.dot"
{{< / highlight >}}

You can compile it with the following command:

    $ ocamlopt -o tree tree.ml

### That's all there is to it




    


