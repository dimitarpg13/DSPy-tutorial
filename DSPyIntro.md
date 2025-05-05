# DSPy Intro

## Introductory Notes

<ins>Open research problem</ins>: Design appropriate prompts for language models (LM) and build pipelines of those to solve complex tasks.

<ins>An issue with existing approaches</ins>: LM pipelines are implemented with hard-coded "prompt templates" which are lengthy strings discovered via trial and error. 
We need a more systematic approach.

<ins>Solution for the "prompt templates" defficiency</ins>: a new programming model that abstracts LM pipelines as _text transformation graphs_ i.e. imperative computation graphs where LMs are invoked through declarative modules.
DSPy modules are _parametrized_, meaning theu can learn by creating and collecting demonstrations how to apply compositions of prompting, finetuning, augmentation, and reasoning techniques. Compiler is designed that will optimize the DSPy pipeline to maximize a given metric.

**DSPy Programming model**:
DSPy pushes building new LM pipelines away from manipulaing free-form strings and moves closer to _programming_ by composing modular operators to build text transformation graphs. In those graphs a compiler automatically generates optimized LM invocaton strategies and prompts from a program. 


