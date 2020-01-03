+++
title = "C++ Dusty Corner number 537: 0 is sometimes a special number"
date = "2014-03-07"
slug = "2014/03/07/c---dusty-corner-number-537-0-is-sometimes-a-special"
Categories = []
+++

I recently discovered the [C++ Quiz site](http://cppquiz.org/) and 
figuring that it's always good to practice C++ skills I started going
through the questions. The third question I 
encountered gave me some pause...  The question was: 
  
    According to the C++11 standard, what is the output of this program?

    #include <iostream>

    void print(char const *str) { std::cout << str; }
    void print(short num) { std::cout << num; }

    int main() {
      print("abc");
      print(0);
      print('A');
    }


Now the obvious response would be "abc065", but I suspected there was some 
sort of trickery afoot here. Finally, I just entered "abc065" and of course
my suspicion was right: the answer was incorrect. Then I went ahead and
"cheated" by pasting the code into a file and compiling it:

    $ g++ -o quiz1 quiz1.cpp
    quiz1.cpp: In function ‘int main()’:
    quiz1.cpp:9:10: error: call of overloaded ‘print(int)’ is ambiguous
    print(0);
          ^
    quiz1.cpp:9:10: note: candidates are:
    quiz1.cpp:4:6: note: void print(const char*)
    void print(char const *str) { std::cout << str; }
    quiz1.cpp:5:6: note: void print(short int)
    void print(short num) { std::cout << num; }

*"Umm... ok"*, I thought *"shouldn't the int literal 0 be cast to a short
automatically? What gives here?"*. Maybe clang will give me a more 
descriptive error message? It's known for that, right?

    $ clang++  -o quiz1 quiz1.cpp
    quiz1.cpp:9:3: error: call to 'print' is ambiguous
    print(0);
          ^~~~~
    quiz1.cpp:4:6: note: candidate function
    void print(char const *str) { std::cout << str; }
    quiz1.cpp:5:6: note: candidate function
    void print(short num) { std::cout << num; }

Ok, so clang didn't reveal any new information here... other than 
those fancy tildas. So I went back to the CPP Quiz site and chose 
"has compilation error" and clicked 'Answer' to get the explanation:

    Sneaky ambiguous function call.

    The statement print(0); is ambiguous due to overload resolution rules. 
    Both print functions are viable, but for the compiler to pick one, 
    one of them has to have a better conversion sequence than the other. 
    §13.3.3¶2: "If there is exactly one viable function that is a better 
    function than all other viable functions, then it is the one selected 
    by overload resolution; otherwise the call is ill-formed".
    
    (a) *Because 0 is a null pointer constant[1], it can be converted 
    implicitly into any pointer type with a single conversion.*
    
    (b) Because 0 is of type int, it can be converted implicitly to a 
    short with a single conversion too.
    
    In our case, both are standard conversion sequences with a single 
    conversion of "conversion rank". Since no function is better than 
    the other, the call is ill-formed.
    
    [1] §4.10¶1 A null pointer constant is an integral constant expression 
    (5.19) prvalue of integer type that evaluates to zero(...) A null 
    pointer constant can be converted to a pointer type.
  
Ooookaaayy... so this is one of those occasions where being a C++ programmer
is very much akin to being a lawyer: you need to be up on all of the 
provisos, caveats and special exemptions in the law (or in the spec in this
case).

So what happened? Passing '0' to the *print* function can interpretted as 
either passing a null pointer to the first *print* function or as a short
0 to the second, overloaded *print* function. Ok, so couldn't any 
integer being passed to the print function also be interpreted as possibly
being a pointer?  So as an experiment I changed: 

    print(0);
To:
    print(2);

Of course, then it compiles just fine. So '0' is a *special* number in this
context because it's also the null pointer constant.

If you are a programming polyglot like me, your first reaction upon realizing 
this is probably to want to run to the relative safety of gated communities
such as OCaml, Haskell or maybe Python where these kinds of incidents 
just don't happen (because no pointers -> no NULL -> no special case for 0). 
...until you realize that those neighborhoods have their own, different 
quirks and in fact there's no perfect language (well, except for Lisp, 
maybe, but *which Lisp?*).

Sometimes you've gotta hang out in the C++ hood with all of the sirens 
and gunshots in the background in order to get things done. As always, you 
just need to be very wary while you're in the C++ hood in order to survive.
Fortunately, in this case it's just a compilation error that seems rather
confusing at first, not a segfault. 

Sure it's definitely strange that 0 is a special case integer in this 
context, but to be fair, how often would you actually run into this 
situation in C++? I'd guess it would be very rare. In most cases you 
wouldn't be passing a '0' literal to a function like that - you'd instead 
be passing in a variable and even if that variable contains '0', that's 
just fine. 

