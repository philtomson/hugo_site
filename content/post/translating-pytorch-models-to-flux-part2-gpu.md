+++
title = "Translating PyTorch models to Flux.jl Part2: Running on GPU"
date  = "2018-06-20"
slug  = "2018/06/20/translating-pytorch-models-to-flux-part2-GPU"
+++

In the [previous post](https://philtomson.github.io/blog/2018-06-15-translating-pytorch-models-to-flux.jl-part1-rnn/) I translated a simple [PyTorch RNN](https://www.cpuheater.com/deep-learning/introduction-to-recurrent-neural-networks-in-pytorch/) to [Flux.jl](http://fluxml.ai/Flux.jl/stable/) a machine learning framework for Julia.

Here's the Julia code modified to use the GPU (and refactored a bit from the previous version; I've put the prediction section into a *predict* function):

{{< highlight julia "linenos=table,linenostart=1" >}}
using Flux
using Flux.Tracker
using Plots
using CuArrays

srand(1)

input_size, hidden_size, output_size = 7, 6, 1
epochs = 200
seq_length = 20
lr = 0.1

data_time_steps = (linspace(2,10, seq_length+1))
data = (sin.(data_time_steps))

x = gpu(data[1:end-1])
y = gpu(data[2:end])

w1 = (param(gpu(randn(input_size,  hidden_size))))
w2 = (param(gpu(randn(hidden_size, output_size))))

function forward(input, context_state, W1, W2)
   #xh = cat(2,input, context_state) 
   #  Due to a Flux bug you have to do:
   xh = gpu(cat(2, Tracker.collect(input), context_state))
   context_state = CUDAnative.tanh.(xh*W1)
   out = context_state*W2
   return out, context_state
end

function train() 
   for i in 1:epochs
      total_loss = 0
      context_state = gpu(zeros(1,hidden_size))
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
        context_state = gpu(context_state.data) 
      end
      if(i % 10 == 0)
        println("Epoch: $i  loss: $total_loss")
      end
   end
end

function predict(x,w1,w2)
   context_state = gpu(zeros(1,hidden_size))
   predictions = []
   for i in 1:length(x)
     input = x[i]
     pred, context_state = forward(input, context_state, w1, w2)
     append!(predictions, pred.data)
   end
   predictions
end

println("train")
train()
println("predict")
predictions = predict(x,w1,w2)
println("... plot ... ")
Plots.backend(:gr)
scatter(data_time_steps[1:end-1], x, label="actual")
scatter!(data_time_steps[2:end],predictions, label="predicted")

{{< / highlight >}}

On line 4 we're now *using CuArrays*. This will pull in CUDA arrays from [CuArrays.jl](https://github.com/JuliaGPU/CuArrays.jl).

Now in order to indicate that we want some data on the GPU we wrap it in the *Flux.gpu()* function as we do for the *x* and *y* assignments on lines 16 & 17. Note that the weights *w1* and *w2* are also tracked parameters for the sake of backpropagation so for those assignments (lines 19 & 20) we call *gpu* on the *randn* matrices and then pass that as the parameter to *param*.

The other signficant change here is that now in the *forward()* function where we call the activation function (line 26) it must chang from:

    context_state = tanh.(xh*W1)

to:


    context_state = CUDAnative.tanh.(xh*W1)

This wasn't well documented in Flux or CuArrays, but without this change I got this rather unhelpful error message:

    ERROR: LoadError: CUDA error: invalid program counter (code #718, ERROR_INVALID_PC)
 

So apparently we need to call the *tanh* defined in [CUDAnative](https://github.com/JuliaGPU/CUDAnative.jl) in order for it to actually run the *tanh* on the GPU.

It's important to note that if you comment out the *using CuArrays* on line 4, the calls to *gpu* don't do anything and the operations will run on the CPU as if those calls weren't there. 

So, how did running this model on the GPU affect performance? It looks like it actually degraded performance. With the GPU enabled (line 4 uncommented):

    real	0m30.648s
    user	0m30.005s
    sys	  0m1.068s

And without the GPU enabled (line 4 commented and line 26 changed to use *tanh()* instead of *CUDAnative.tanh()*):

    real	0m17.032s
    user	0m17.091s
    sys	  0m0.452s

I think this is because this is a fairly small model and the overhead of passing data to/from the GPU is greater than the computational overhead.

I do kind of wish we could switch between GPU and CPU more easily - some of the Python frameworks seem to make this easier. Calls to *CUDAnative.op()* have to be changed back to *op()* when running on the CPU (as with the *tanh()* call above). Perhaps some change in CuArrays.jl could get around this such that if you're *using CuArrays* you get *CUDAnative.op()* instead of *op()* when the operands are *CuArrays*?

    

