+++
title = "Translating PyTorch models to Flux.jl Part1: RNN"
date  = "2018-06-15"
slug  = "2018/06/15/translating-pytorch-models-to-flux-part1-rnn"
+++

[Flux.jl](http://fluxml.ai/Flux.jl/stable/) is a machine learning framework built in Julia. It has some similarities to PyTorch, and like most modern frameworks includes autodifferentiation. It's definitely still a work in progress, but it is being actively developed (including several GSoC projects this summer).

I was curious about how easy/difficult it might be to convert a PyTorch model into Flux.jl. I found a fairly simple [PyTorch tutorial on RNNs](https://www.cpuheater.com/deep-learning/introduction-to-recurrent-neural-networks-in-pytorch/) to translate. My goal here isn't to explain RNNs (see the [linked article](https://www.cpuheater.com/deep-learning/introduction-to-recurrent-neural-networks-in-pytorch/) for that) - my intent is to see what is required to go from the PyTorch/Python ecosystem to the Flux.jl/Julia ecosystem.

Let's start with the PyTorch code:

{{< highlight python "linenos=table,linenostart=1" >}}
import torch
from torch.autograd import Variable
import numpy as np
import pylab as pl
import torch.nn.init as init


torch.manual_seed(1)

dtype = torch.FloatTensor
input_size, hidden_size, output_size = 7, 6, 1
epochs = 200
seq_length = 20
lr = 0.1

data_time_steps = np.linspace(2, 10, seq_length + 1)
data = np.sin(data_time_steps)
data.resize((seq_length + 1, 1))

x = Variable(torch.Tensor(data[:-1]).type(dtype), requires_grad=False)
y = Variable(torch.Tensor(data[1:]).type(dtype), requires_grad=False)

w1 = torch.FloatTensor(input_size, hidden_size).type(dtype)
init.normal(w1, 0.0, 0.4)
w1 =  Variable(w1, requires_grad=True)
w2 = torch.FloatTensor(hidden_size, output_size).type(dtype)
init.normal(w2, 0.0, 0.3)
w2 = Variable(w2, requires_grad=True)

def forward(input, context_state, w1, w2):
  xh = torch.cat((input, context_state), 1)
  context_state = torch.tanh(xh.mm(w1))
  out = context_state.mm(w2)
  return  (out, context_state)

for i in range(epochs):
  total_loss = 0
  context_state = Variable(torch.zeros((1, hidden_size)).type(dtype), requires_grad=True)
  for j in range(x.size(0)):
    input = x[j:(j+1)]
    target = y[j:(j+1)]
    (pred, context_state) = forward(input, context_state, w1, w2)
    loss = (pred - target).pow(2).sum()/2
    total_loss += loss
    loss.backward()
    w1.data -= lr * w1.grad.data
    w2.data -= lr * w2.grad.data
    w1.grad.data.zero_()
    w2.grad.data.zero_()
    context_state = Variable(context_state.data)
  if i % 10 == 0:
     print("Epoch: {} loss {}".format(i, total_loss.data[0]))


context_state = Variable(torch.zeros((1, hidden_size)).type(dtype), requires_grad=False)
predictions = []

for i in range(x.size(0)):
  input = x[i:i+1]
  (pred, context_state) = forward(input, context_state, w1, w2)
  context_state = context_state
  predictions.append(pred.data.numpy().ravel()[0])


pl.scatter(data_time_steps[:-1], x.data.numpy(), s=90, label="Actual")
pl.scatter(data_time_steps[1:], predictions, label="Predicted")
pl.legend()
pl.show()

{{< / highlight >}}

And now the Flux.jl version in Julia:

{{< highlight julia "linenos=table,linenostart=1" >}}
using Flux
using Flux.Tracker
using Plots

srand(1)

input_size, hidden_size, output_size = 7, 6, 1
epochs = 200
seq_length = 20
lr = 0.1

data_time_steps = linspace(2,10, seq_length+1)
data = sin.(data_time_steps)

x = data[1:end-1]
y = data[2:end]

w1 = param(randn(input_size,  hidden_size))
w2 = param(randn(hidden_size, output_size)) 

function forward(input, context_state, W1, W2)
   #xh = cat(2,input, context_state) 
   #  Due to a Flux bug you have to do:
   xh = cat(2, Tracker.collect(input), context_state) 
   context_state = tanh.(xh*W1)
   out = context_state*W2
   return out, context_state
end

function train() 
   for i in 1:epochs
      total_loss = 0
      context_state = param(zeros(1,hidden_size))
      for j in 1:length(x)
        input = x[j]
        target= y[j]
        pred, context_state = forward(input, context_state, w1, w2)
        loss = sum((pred .- target).^2)/2
        total_loss += loss
        back!(loss)
        w1.data .-= lr.*w1.grad
        w2.data .-= lr.*w2.grad
        w1.grad .= 0.0
        w2.grad .= 0.0 
        context_state = param(context_state.data)
      end
      if(i % 10 == 0)
        println("Epoch: $i  loss: $total_loss")
      end
   end
end

train()

context_state = param(zeros(1,hidden_size))
predictions = []

for i in 1:length(x)
  input = x[i]
  pred, context_state = forward(input, context_state, w1, w2)
  append!(predictions, pred.data)
end
scatter(data_time_steps[1:end-1], x, label="actual")
scatter!(data_time_steps[2:end],predictions, label="predicted")
{{< / highlight >}}

The Flux.jl/Julia version is very similar to the PyTorch/Python version.
A few notable differences:

* Numpy functionality is builtin to Julia. No need to import numpy. 
* torch.Variable maps to Flux.param
* x and y are type torch.Variable in the PyTorch version, while they're just regular builtin matrices on the Julia side.
* Flux.param(var) indicates that the variable *var* will be tracked for the purposes of determining gradients (just as torch.Variable).
* I did run into a bug in Flux.jl; you'll notice the workaround on line 24. Ultimately, when the bug is fixed you'll be able to uncomment line 22 and eliminate line 24. The bug had to do with how certain tracked collections are translated to scalar types. The tracking is required for back propagation and the problem was that the *input* being passed into the *foward* function would get another level of unnecessary tracking each time *forward* was called.
* '.' prior to an operator (such as at line 41 in the Julia code) indicates a broadcasting operation in Julia. Note also line '.' after the tanh at line 25, it indicates that the tanh is broadcast to the matrix that results from xh\*W1. (From what I can tell, numpy sort of automatically determines whether an operation should broadcast or not based on the dimensions of the operands - Julia is more explicit about this.)
* Even the plotting at the end is very similar between the two versions.

In the next post I'll modify the Julia version to use the GPU.

