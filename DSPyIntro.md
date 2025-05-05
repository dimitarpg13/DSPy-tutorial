# DSPy Intro

## Introductory Notes

<ins>Open research problem</ins>: Design appropriate prompts for language models (LM) and build pipelines of those to solve complex tasks.

<ins>An issue with existing approaches</ins>: LM pipelines are implemented with hard-coded "prompt templates" which are lengthy strings discovered via trial and error. 
We need a more systematic approach.

<ins>Solution for the "prompt templates" defficiency</ins>: a new programming model that abstracts LM pipelines as _text transformation graphs_ i.e. imperative computation graphs where LMs are invoked through declarative modules.
DSPy modules are _parametrized_, meaning theu can learn by creating and collecting demonstrations how to apply compositions of prompting, finetuning, augmentation, and reasoning techniques. Compiler is designed that will optimize the DSPy pipeline to maximize a given metric.

**DSPy Programming model**:
DSPy pushes building new LM pipelines away from manipulaing free-form strings and moves closer to _programming_ by composing modular operators to build text transformation graphs. In those graphs a compiler automatically generates optimized LM invocaton strategies and prompts from a program. 
Implementing DSPy programming model involves the following steps: 
1) translate the string-based prompting technqiues, including complex and task-dependent ones like Chain of Thought [1] and ReAct [2] into declarative modules that carry _natural-language typed signatures_. 
DSPy modules are task-adaptive components - akin to neural network layers - that abstract any particular text transformation, like answering a question or summarizing a paper.
2) parametrize each DSPy module so that it can _learn_ its desried behavior by iteratively bootstrapping useful demonstrations within the pipeline.
3) DSPy modules are used via expressive _define-by-run_ computational graphs (this is inspired by PyTorch abstractions)
4) Pipelines are expressed by (i) declaring the needed modules and (ii) using these modules in any logical control flow (e.g. `if` statements, `for` loops, exceptions) to logically conect the modules

**DSPy Compiler**:



## References

[1] [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models, Jason Wei et al, 2023](https://github.com/dimitarpg13/DSPy-tutorial/blob/main/docs/Chain-of-Thought_Prompting_Elicits_Reasoning_in_Large_Language_Models_Wei_2022.pdf) 

[2] [ReAct: Synergizing Reasoning and Acting in Language Models, S. Yao et al, 2023](https://github.com/dimitarpg13/DSPy-tutorial/blob/main/docs/ReAct-Synergizing_Reasoning_and_Acting_in_Language_Models_Yao_2022.pdf)

