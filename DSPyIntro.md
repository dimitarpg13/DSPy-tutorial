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

DSPy compiler optimizes the DSPy program to improve quality and cost. The compiler inputs are the program, few training inputs with optional labels, and a validation metric. The compiler simulates versions of the program on the inputs and _bootstraps_ example traces of each module for self-improvement, using them to construct effective few-shot prompts or finetuning small LMs for steps of the pipeline.

**Optimization in DSPy**:

Modular, conducted by _teleprompters_ which are gen purpose optimization strategies that determine how the modules should learn from data. In this way, the compiler automatically maps the declarative modules to _high-quality_ compositions of prompting, finetuning, reasoning, and augmentation.

## Details on DSPy Programming Model

DSPy treats LMs as abstract devices for text generation, and optimizes their usage in arbitrary computation graphs. DSPy programs are expressed via Python code: each program takes the task input (e.g. a question to answer or a paper to summarize) and returns the output (e.g. an answer or a summary) after a series of steps. DSPy contributes three abstractions toward automatic optimization: 

* **signatures**
* **modules**
* **teleprompters**

Signatures abstract the input/output behavior of a module; Modules replace existing hand-prompting technqiues and can be composed in arbitrary pipelines; Teleprompters optimize all modules in the pipeline to maximize specified metric.

Instead of free-form string prompts, DSPy programs use natural language _signatures_ to assign work to the LM. A DSPy signature is _natural-language typed_ declaration of a function: a short declarative spec that tells DSPy _what_ a text transformation needs to do (e.g. "consume questions and return answers"), rather than _how_ a specific LM should be prompted to implement that behavior. More formally, a DSPy signature is a tuple of _input fields_ and _output fields_ (and an optional _instruction_). A field consists of a _field name_ and optional metadata. In typical usage, the roles of fields are inferred by DSPy as a function of field names. For instance, the DSPy compiler will use in-context learning to inerpret `question` differently from `answer` and will iteratively refine its usage of these fields.

Signatures offer two benefits over prompts: they can be compiled into self-improving and pipeline-adaptive prompts or finetunes. This is primarily done by _bootstrapping_ useful demo exampkles for each signature. Additionally, they handle structured formatting and parsing logic to reduce (or, ideally, avoid) brittle string manipulation in user programs.

In practice, DSPy signatures can be expressed with a shorthand notation like `question -> answer`, so that line 1 in the following is a complete DSPy program for a basic question-answering system:

```python
qa = dspy.Predict("question -> answer")
qa(question="Where is Guarani spoken?")
# Out: Prediction(answer="Guarani is spoken mainly in South America.")
```
In the shorthand notation, each field's name indicates the semantic role that the input (or output) field plays in the transmission. DSPy will parse this notation and expand the field names into meaningful instructions for the LM, so that `english_document -> french_translation` would prompt for English to French translation. When needed, DSPy can offer more advanced programming interfaces for expressing more explicit constraints on signatures.

### Parametrized and templated modules

Parametrized and templated modules can abstract prompting techniques. Similarly to type signatures in programming languages, DSPy signatures simply define an interface and provide type-like hints on the expected behavior. To use a signature, we must declare a _module_ with that signature, like we instantiated a `Predict` module above. A module declaration like this returns a _function_ having that signature. 

**The `Predict` Module**
The core module for working with signatures in DSPy is `Predict`. Internally, `Predict` stores the supplied signature, an optional LM to use (initially `None`, nut otherwise the default LM for this module), and a list of demonstrations for prompting (initially empty). 
Like layers in PyTorch, the instantiated module behaves like a callable function: it takes in keyword arguments corresponding to the signature input fields (e.g. `question`), formats a prompt to implement the signature and includes the appropriate deomnstrations, calls the LM, and parses the output fields. When `Predict` detects it is being used in `compile` mode, it will also internally track input/output traces to assist the teleprompter at bootstrapping the demonstrations.

**Other Built-in Modules**
DSPy modules translate prompting techniques into modular functions that support any signature, contrasting with the standard approach of prompting LMs with task-specific details (e.g. hand-written few-shot examples). To this end, DSPy includes a number of more sophisticated modules like `ChainOfThought`, `ProgramOfThought`, `MultiChainComparison` and `ReAct`. These can all be used interchangeably to implement a DSPy signature. For instance, simply changing `Predict` to `ChainOfThought` in the above program leads to a system that thinks step by step before committing to its output field. 

Importantly, all of these modules are implemented in a few lines of code by expanding the user-defined signature and calling `Predict` one or more times on new signatures as appropriate. 
For instance, we show a simplified implementaton of the built-on `ChainOfThought` below.

```python
class ChainOfThought(dspy.Module):
   def __init__(self, signature):
       # modify signature from '*inputs -> *outputs' to '*inputs -> rationale, *outputs'
       rationale_field = dspy.OutputField(prefix='Reasoning: Let's think step by step.')
       signature = dspy.Signature(signature).prepend_output_field(rationale_field)

       # declare a sub-module with the modified signature
       self.predict = dspy.Predict(signature)

   def forward(self, **kwargs):
       # just forward the inputs to the sub-module
       return self.predict(**kwargs)
```

This is a fuly-fledged module capable of learning effective few-shot prompting for any LM or task.

**Parametrization** 
DSPy _parametrizes_ the set of prompting tehcniques which it implements in modules. To understand this parametrization, observe that any LM call seeking to implement a particular signature needs to specify _parameters_ that include: 

(1) the specific LM to call
(2) the prompt instructions and the string prefix for each signature field
(3) the demonstrations used as few-shot prompts (for frozen LMs) or as training data (for finetuning)

We focus primarily on automatically generating and selecting useful demonstrations. In our case studies, we find that bootstrapping good demonstrations gives us a powerful way to teach sophisticated pipelines of LMs new behaviors systematically.

**Tools** 
DSPy proramns may use tools which are modules that execute computation.
DSPy supports retrieval models through a `dspy.Retrieve` module.  There are `dspy.SQL` for executing SQL queries and `dspy.PythonInterpreter` for executing Python code in a sandbox.

**Programs**
DSPy modules can be composed in arbitrary pipelines in a define-by-run interface. Inspired directly by PyTorch one first declares the modules needed at initialization, allowing DSPy to keep track of them for optimization, and then one expresses the pipeline with arbitrary code that calls the modules in a `forward` method. As a simple illustration, we offer the following simple but complete retrieval-augmented generation (RAG) system.

```python
class RAG(dspy.Module):
   def __init__(self, num_passages=3):
      # `Retrieve` will use user's default retrieval settings unless overridden.
      self.retrieve = dspy.Retrieve(k=num_passages)
      # `ChainOfThought` with signature that generates answers given retrieval & question.
      self.generate_answer = dspy.ChainOfThought("context, question -> answer")

   def forward(self, question):
      context = self.retrieve(question).passages
      return self.generate_answer(context=context, question=question) 
```
To highlight modularity, we use `ChainOfThought` as a drop-in replacement of the basic `Predict`. One can now simply write `RAG()("Where is Guarani spoken?")` to use it. Notice that, if we use a signature `"context, question -> search_query"`, we get a system that generates search queries rather than answers.



## References

[1] [Chain-of-Thought Prompting Elicits Reasoning in Large Language Models, Jason Wei et al, 2023](https://github.com/dimitarpg13/DSPy-tutorial/blob/main/docs/Chain-of-Thought_Prompting_Elicits_Reasoning_in_Large_Language_Models_Wei_2022.pdf) 

[2] [ReAct: Synergizing Reasoning and Acting in Language Models, S. Yao et al, 2023](https://github.com/dimitarpg13/DSPy-tutorial/blob/main/docs/ReAct-Synergizing_Reasoning_and_Acting_in_Language_Models_Yao_2022.pdf)

