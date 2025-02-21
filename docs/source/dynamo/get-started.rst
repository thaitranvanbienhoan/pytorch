Getting Started
===============

Let’s start with a simple example and make things more complicated step
by step. Please note that you’re likely to see more significant speedups
the newer your GPU is.

.. code:: python

   from torch._dynamo import optimize
   import torch
   def fn(x, y):
       a = torch.cos(x).cuda()
       b = torch.sin(y).cuda()
       return a + b
   new_fn = optimize("inductor")(fn)
   input_tensor = torch.randn(10000).to(device="cuda:0")
   a = new_fn()

This example will not actually run faster. Its purpose is to demonstrate
the ``torch.cos()`` and ``torch.sin()`` features which are
examples of pointwise ops as in they operate element by element on a
vector. A more famous pointwise op you might actually want to use would
be something like ``torch.relu()``. Pointwise ops in eager mode are
suboptimal because each one would need to need to read a tensor from
memory, make some changes and then write back those changes. The single
most important optimization that inductor does is fusion. So back to our
example we can turn 2 reads and 2 writes into 1 read and 1 write which
is crucial especially for newer GPUs where the bottleneck is memory
bandwidth (how quickly you can send data to a GPU) instead of compute
(how quickly your GPU can crunch floating point operations)

Another major optimization that inductor makes available is automatic
support for CUDA graphs.
CUDA graphs help eliminate the overhead from launching individual
kernels from a python program which is especially relevant for newer GPUs.

dynamo supports many different backends but inductor specifically works
by generating `Triton <https://github.com/openai/triton>`__ kernels and
we can inspect them by running ``TORCHINDUCTOR_TRACE=1 python trig.py``
with the actual generated kernel being

.. code:: python

   @pointwise(size_hints=[16384], filename=__file__, meta={'signature': {0: '*fp32', 1: '*fp32', 2: 'i32'}, 'device': 0, 'constants': {}, 'configs': [instance_descriptor(divisible_by_16=(0, 1, 2), equal_to_1=())]})
   @triton.jit
   def kernel(in_ptr0, out_ptr0, xnumel, XBLOCK : tl.constexpr):
       xnumel = 10000
       xoffset = tl.program_id(0) * XBLOCK
       xindex = xoffset + tl.reshape(tl.arange(0, XBLOCK), [XBLOCK])
       xmask = xindex < xnumel
       x0 = xindex
       tmp0 = tl.load(in_ptr0 + (x0), xmask)
       tmp1 = tl.sin(tmp0)
       tmp2 = tl.sin(tmp1)
       tl.store(out_ptr0 + (x0 + tl.zeros([XBLOCK], tl.int32)), tmp2, xmask)

And you can verify that fusing the two ``sins`` did actually occur
because the two ``sin`` operations occur within a single Triton kernel
and the temporary variables are held in registers with very fast access.

You can read up a lot more on Triton’s performance
`here <https://openai.com/blog/triton/>`__ but the key is it’s in python
so you can easily understand it even if you haven’t written all that
many CUDA kernels.

As a next step let’s try a real model like resnet50 from the PyTorch
hub.

.. code:: python

   import torch
   import torch._dynamo as dynamo
   model = torch.hub.load('pytorch/vision:v0.10.0', 'resnet18', pretrained=True)
   opt_model = dynamo.optimize("inductor")(model)
   model(torch.randn(1,3,64,64))

And that’s not the only available backend, you can run in a REPL
``dynamo.list_backends()`` to see all the available ones. Try out the
``aot_cudagraphs`` or ``nvfuser`` next as inspiration.

Let’s do something a bit more interesting now, our community frequently
uses pretrained models from
`transformers <https://github.com/huggingface/transformers>`__ or
`TIMM <https://github.com/rwightman/pytorch-image-models>`__ and one of
our design goals is for dynamo and inductor to work out of the box with
any model that people would like to author.

So we’re going to directly download a pretrained model from the
HuggingFace hub and optimize it:

.. code:: python

   import torch
   from transformers import BertTokenizer, BertModel
   import torch._dynamo as dynamo
   # Copy pasted from here https://huggingface.co/bert-base-uncased
   tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
   model = BertModel.from_pretrained("bert-base-uncased").to(device="cuda:0")
   model = dynamo.optimize("inductor")(model) # This is the only line of code that we changed
   text = "Replace me by any text you'd like."
   encoded_input = tokenizer(text, return_tensors='pt').to(device="cuda:0")
   output = model(**encoded_input)

If you remove the ``to(device="cuda:0")`` from the model and
encoded_input then triton will generate C++ kernels that will be
optimized for running on your CPU. You can inspect both Triton or C++
kernels for BERT, they’re obviously more complex than the trigonometry
example we had above but you can similarly skim it and understand if you
understand PyTorch.

Similarly let’s try out a TIMM example

.. code:: python

   import timm
   import torch._dynamo as dynamo
   import torch
   model = timm.create_model('resnext101_32x8d', pretrained=True, num_classes=2)
   opt_model = dynamo.optimize("inductor")(model)
   opt_model(torch.randn(64,3,7,7))

Our goal with dynamo and inductor was to build the highest coverage ML compiler which should work with any model you throw at it.

Existing Backends
~~~~~~~~~~~~~~~~~

TorchDynamo has a growing list of backends, which can be found in
`backends.py <https://github.com/pytorch/pytorch/blob/master/torch/_dynamo/optimizations/backends.py>`__
or ``torchdynamo.list_backends()`` each of which with its optional dependencies.

Some of the most commonly used backend include:

* **Debugging backends**: \* ``dynamo.optimize("eager")`` - Uses PyTorch
  to run the extracted GraphModule. This is quite useful in debugging
  TorchDynamo issues. \* ``dynamo.optimize("aot_eager")`` - Uses
  AotAutograd with no compiler, i.e, just using PyTorch eager for the
  AotAutograd’s extracted forward and backward graphs. This is useful for
  debugging, and unlikely to give speedups.

* **Training & inference backends**: \* ``dynamo.optimize("inductor")`` -
  Uses TorchInductor backend with AotAutograd and cudagraphs by leveraging
  codegened Triton kernels `Read
  more <https://dev-discuss.pytorch.org/t/torchinductor-a-pytorch-native-compiler-with-define-by-run-ir-and-symbolic-shapes/747>`__

  * ``dynamo.optimize("nvfuser")`` - nvFuser with TorchScript. `Read more <https://dev-discuss.pytorch.org/t/tracing-with-primitives-update-1-nvfuser-and-its-primitives/593>`__

  * ``dynamo.optimize("aot_nvfuser")`` - nvFuser with AotAutograd. `Read more <https://dev-discuss.pytorch.org/t/tracing-with-primitives-update-1-nvfuser-and-its-primitives/593>`__

  * ``dynamo.optimize("aot_cudagraphs")`` - cudagraphs with AotAutograd. `Read more <https://github.com/pytorch/torchdynamo/pull/757>`__

* **Inference-only backend**\ s: \* ``dynamo.optimize("ofi")`` - Uses
  Torchscript optimize_for_inference. `Read
  more <https://pytorch.org/docs/stable/generated/torch.jit.optimize_for_inference.html>`__

  * ``dynamo.optimize("fx2trt")`` - Uses Nvidia TensorRT for inferenc optimizations. `Read more <https://github.com/pytorch/TensorRT/blob/master/docsrc/tutorials/getting_started_with_fx_path.rst>`__

  * ``dynamo.optimize("onnxrt")`` - Uses ONNXRT for inference on CPU/GPU. `Read more <https://onnxruntime.ai/>`__ \* ``dynamo.optimize("ipex")`` - Uses IPEX for inference on CPU. `Read more <https://github.com/intel/intel-extension-for-pytorch>`__

Why do you need another way of optimizing PyTorch code?
-------------------------------------------------------

While a number of other code optimization tools exist in the PyTorch
ecosystem, each of them has its own flow. Here is a few examples of
existing methods and their limitations:

-  ``torch.jit.trace()`` is silently wrong if it cannot trace e.g:
   during control flow
-  ``torch.jit.script()`` requires modifications to user or library code
   by adding type annotations and removing non PyTorch code
-  ``torch.fx.symbolic_trace()`` either traces correctly or gives a hard
   error but it’s limited to traceable code so still can’t handle
   control flow
-  ``torch._dynamo`` works out of the box and produces partial graphs.
   It still has the option of producing a single graph with
   ``nopython=True`` which are needed for `some
   situations <./documentation/FAQ.md#do-i-still-need-to-export-whole-graphs>`__
   but allows a smoother transition where partial graphs can be
   optimized without code modification

.. |image0| image:: ../_static/img/dynamo/TorchDynamo.png
