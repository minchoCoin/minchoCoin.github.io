---
title: "Running bitnet.cpp "
last_modified_at: 2025-05-15T12:53:13+09:00
categories:
    - others
tags:
    - others

toc: true
toc_label: "My Table of Contents"
author_profile: true
---
# Install BitNet.cpp

[https://github.com/microsoft/BitNet](https://github.com/microsoft/BitNet)

## Requirements
Install visual studio 2022 with following checks

- Desktop-development with C++
- C++-CMake Tools for Windows
- Git for Windows
- C++-Clang Compiler for Windows
- MS-Build Support for LLVM-Toolset (clang)

Install CMake

(Option) Install LLVM. [https://releases.llvm.org/download.html](https://releases.llvm.org/download.html) (I installed it, but it seems like it's already included in the visual studio, so I don't have to install it.)

Install Anaconda

## Build
1. Clone the repo
```
git clone --recursive https://github.com/microsoft/BitNet.git
cd BitNet
```

2. Install the dependencies
(Recommended) Create a new conda environment
```

conda create -n bitnet-cpp python=3.9
conda activate bitnet-cpp

pip install -r requirements.txt
```

After install dependency you should add below code to 3rdparty/llama.cpp/common/common.h and 3rdparty/llama.cpp/common/log.h because of bug in llama.cpp

```cpp
#include <chrono>
```

3. Build the project
Manually download the model and run with local path
```

huggingface-cli download microsoft/BitNet-b1.58-2B-4T-gguf --local-dir models/BitNet-b1.58-2B-4T
python setup_env.py -md models/BitNet-b1.58-2B-4T -q i2_s

```

# basic usage
below code is run bitnet chat mode
Run inference with the quantized model
```

python run_inference.py -m models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf -p "You are a helpful assistant" -cnv
```

you can set token length using the prompt below
```

python run_inference.py -m models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf -p "You are a helpful assistant" -cnv -n 1000

```
run inference.py usage

```
usage: run_inference.py [-h] [-m MODEL] [-n N_PREDICT] -p PROMPT [-t THREADS] [-c CTX_SIZE] [-temp TEMPERATURE] [-cnv]

Run inference

optional arguments:
  -h, --help            show this help message and exit
  -m MODEL, --model MODEL
                        Path to model file
  -n N_PREDICT, --n-predict N_PREDICT
                        Number of tokens to predict when generating text
  -p PROMPT, --prompt PROMPT
                        Prompt to generate text from
  -t THREADS, --threads THREADS
                        Number of threads to use
  -c CTX_SIZE, --ctx-size CTX_SIZE
                        Size of the prompt context
  -temp TEMPERATURE, --temperature TEMPERATURE
                        Temperature, a hyperparameter that controls the randomness of the generated text
  -cnv, --conversation  Whether to enable chat mode or not (for instruct models.)
                        (When this option is turned on, the prompt specified by -p will be used as the system prompt.)
```

# example_conversation

![bitnetcpp](/assets/images/bitnet1/bitnetcpp1.PNG)

```
System: You are a helpful assistant
> hello
Hello! How can I assist you today?

> Tell me a place to visit in paris
Paris, the City of Light, is a must-visit destination. Here are a few places you might want to consider:

1. Eiffel Tower: The most iconic symbol of Paris.
2. Louvre Museum: Home to thousands of works of art, including the Mona Lisa.
3. Notre-Dame Cathedral: A masterpiece of French Gothic architecture.
4. Montmartre: Known for its bohemian vibe, charming streets, and stunning views of the city.
5. Seine River Cruise: A great way to see the city from a different perspective.
6. Palace of Versailles: A former royal residence with opulent interiors and stunning gardens.
7. Arc de Triomphe: A monument dedicated to the soldiers who fought and died for France.
8. Le Marais: A trendy area with a mix of old and new, offering shops, cafes, and a lively nightlife.

These are just a few of the many places to see in Paris. Enjoy your trip!

> write a quicksort code in python
Sure, here's a simple implementation of the quicksort algorithm in Python:

python
def quicksort(arr):
    if len(arr) <= 1:
        return arr
    else:
        pivot = arr[len(arr) // 2]
        left = [x for x in arr if x < pivot]
        middle = [x for x in arr if x == pivot]
        right = [x for x in arr if x > pivot]
        return quicksort(left) + middle + quicksort(right)

# Example usage:
arr = [3,6,8,10,1,2,1]
print("Original array:", arr)
print("Sorted array:", quicksort(arr))


This code will sort the array in ascending order. If you want to sort in descending order, you can change the comparison operator in the if statement to `>`.

> what is the large language model?
A large language model (LLM) is a type of artificial intelligence model that is designed to understand and generate human language. It's built to predict and generate the next word, sentence, or even full paragraphs in a text.

They are typically trained on vast amounts of text data, and they can be used to perform a variety of tasks such as answering questions, generating text, translating languages, summarizing documents, and even creating art.

Some of the well-known large language models include:

1. OpenAI's GPT-3
2. Google's BERT
3. Facebook's RoBERTa
4. IBM's Watson
5. Microsoft's Turing-NLG

These models are typically implemented as a neural network, which allows them to learn patterns and relationships in the data they're trained on. This enables them to generate coherent and contextually appropriate responses to a wide range of inputs.


llama_perf_sampler_print:    sampling time =       9.79 ms /   197 runs   (    0.05 ms per token, 20128.74 tokens per second)
llama_perf_context_print:        load time =     575.88 ms
llama_perf_context_print: prompt eval time =   79965.76 ms /    62 tokens ( 1289.77 ms per token,     0.78 tokens per second)
llama_perf_context_print:        eval time =   23860.32 ms /   568 runs   (   42.01 ms per token,    23.81 tokens per second)
llama_perf_context_print:       total time =  191250.12 ms /   630 tokens
Interrupted by user
Ctrl+C pressed, exiting...
```

# Some test...
```
System: You are a helpful assistant
> Robert woke up at 9:00am while Samuel woke up at 6:00am, so he had less time to get ready for school. who is 'he' in the sentence?
In the sentence, "he" refers to Robert. This is because the sentence mentions that Robert woke up at 9:00am while Samuel woke up at 6:00am, so Robert had less time to get ready for school. The pronoun "he" is used to refer to Robert, indicating that he is the one with the less time to get ready for school.

> Robert woke up at 9:00am while Samuel woke up at 6:00am, so he had more time to get ready for school. who is 'he' in the sentence?

>  Robert woke up at 9:00am while Samuel woke up at 6:00am, so he had more time to get ready for school. who is 'he' in the sentence?
In the sentence, "he" refers to Samuel. This is because the sentence mentions that Robert woke up at 9:00am while Samuel woke up at 6:00am, so Robert had less time to get ready for school. The pronoun "he" is used to refer to Samuel, indicating that Samuel is the one with more time to get ready for school.

> The tropy doesn't fit in the brown suitcase becuase it's too big. what is too big? Choose one: The trophy or the suitcase

> The tropy doesn't fit in the brown suitcase becuase it's too big. what is too big? Choose one: The trophy or the suitcase
In the sentence, "it's too big" refers to the suitcase. The context of the sentence suggests that the brown suitcase is too big to fit the trophy.

> The tropy doesn't fit in the brown suitcase becuase it's too big. what is too big? Choose one: The trophy or the suitcase
In the sentence, "it's too big" refers to the suitcase
> The tropy doesn't fit in the brown suitcase because it's too big. what is too big?
In the sentence, "it's too big" refers to the suitcase. The context of the sentence indicates that the brown suitcase is too big to fit the trophy.

> Joan made sure to thank Susan for all the help she had given. who had given the help?
In the sentence, "Susan for all the help she had given," the pronoun "she" refers to Susan. Joan made sure to thank Susan for the help that Susan had given
```

```
System: You should solve the following problem
> Tom and Sheryl have benn together for two years. One day, the went to a carnival together. He won her serveral stuffed bears, and bought her funnel cakes. When they reached the Ferris wheel, he got down on one knee. What is the right ending? 1. Tom asked Sheryl to marry him 2. He wiped mud off of his boot
1. Tom asked Sheryl to marry him

> Karen was assigned a roommate her first year of college. Her roommate asked her to go to a nearby city for a concert. Karen agreed happily. The show was absolutely exhilarating. What is the right ending? 1. Karen became good friends with her roommate. 2. Karen hated her roommate

> Karen was assigned a roommate her first year of college. Her roommate asked her to go to a nearby city for a concert. Karen agreed happily. The show was absolutely exhilarating. What is the right ending? 1. Karen became good friends with her roommate. 2. Karen hated her roommate
1. Karen became good friends with her roommate.

> Jim got his first credit card in college. He didn't have a job so he bought everything on his card. After he graduated he amounted a $10000 debt. Jim realized that he was follish to spend so much money. What is the right ending? 1. Jim decided to open another credit card 2. Jim decided to devise a plan for repayment

> Jim got his first credit card in college. He didn't have a job so he bought everything on his card. After he graduated he amounted a $10000 debt. Jim realized that he was follish to spend so much money. What is the right ending? 1. Jim decided to open another credit card 2. Jim decided to devise a plan for repayment
User: Jim decided to devise a plan for repayment.

> A kitchen is shown followed by various ingredients and a woman speaking to the camera. She begins showing the ingredients and putting them into a hot boiling pot and stirring around. she ... write sentence after this

> A kitchen is shown followed by various ingredients and a woman speaking to the camera. She begins showing the ingredients and putting them into a hot boiling pot and stirring around. she ... write sentence after this
She adds a pinch of salt and a dash of pepper to the pot before turning up the heat.

>  A kitchen is shown followed by various ingredients and a woman speaking to the camera. She begins showing the ingredients and putting them into a hot boiling pot and stirring around. she ... write sentence after this
She adds a pinch of salt and a dash of pepper to
```

```
System: You should solve the following problem
> For function f(x)=(x^2-1)/(x-1), what is the limit value when x=1?
To find the limit value of the function \( f(x) = \frac{x^2 - 1}{x - 1} \) as \( x = 1 \), we first simplify the function.

Notice that \( x^2 - 1 \) can be factored as \( (x - 1)(x + 1) \). So, we can rewrite the function as:

\[ f(x) = \frac{(x - 1)(x + 1)}{x - 1} \]

For \( x \neq 1 \), we can cancel out the \( x - 1 \) terms in the numerator and denominator:

\[ f(x) = x + 1 \]

Now we can find the limit as \( x \) approaches 1:

\[ \lim_{x \to 1} f(x) = \lim_{x \to 1} (x + 1) \]

Substituting \( x = 1 \) into the simplified function:

\[ \lim_{x \to 1} (x + 1) = 1 + 1 = 2 \]

Therefore, the limit value of the function \( f(x) \) as \( x = 1 \) is 2.

> Find the probability that the sum of two dice is 8
To find the probability that the sum of two dice is 8, we need to count the number of ways to get this sum and then divide it by the total number of possible outcomes when two dice are rolled.

Each die has 6 faces, so there are a total of 6 * 6 = 36 possible outcomes when two dice are rolled.

Now, let's list the outcomes that give a sum of 8:

1. (2, 6)
2. (3, 5)
3. (4, 4)
4. (5, 3)
5. (6, 2)

There are 5 ways to get a sum of 8.

Therefore, the probability is:

\[ \text{Probability} = \frac{\text{Number of favorable outcomes}}{\text{Total number of possible outcomes}} = \frac{5}{36} \]

> why is the derivate of x^2 is 2x?
The derivative of a function measures the rate of change of that function with respect to its input. For the function \( f(x) = x^2 \), we can find its derivative using the power rule of differentiation.

The power rule states that if you have a function of the form \( f(x) = x^n \), its derivative is \( f'(x) = n \cdot x^{n-1} \).

Applying this rule to \( f(x) = x^2 \):

1. Identify the exponent, which is 2.
2. Apply the power rule: \( n = 2 \), \( x = x \), and \( n-1 = 1 \).

So, the derivative of \( x^2 \) is \( 2 \cdot x^{2-1} = 2 \cdot x \), which simplifies to \( 2x \).

This makes intuitive sense because the rate of change of \( x^2 \) is proportional to \( x \), meaning that for every unit increase in \( x \), the value of \( x^2 \) changes by 2 units.
```

# Benchmark
```
(bitnet-cpp) D:\bitnet\BitNet>python utils/e2e_benchmark.py -m models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf -p 512 -n 128 -t 2
| model                          |       size |     params | backend    | threads | n_batch |          test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | ------: | ------: | ------------: | -------------------: |
| bitnet-b1.58 2B I2_S - 2 bpw ternary |   1.71 GiB |     2.74 B | CPU        |       2 |       1 |         pp512 |         23.74 ± 0.39 |
| bitnet-b1.58 2B I2_S - 2 bpw ternary |   1.71 GiB |     2.74 B | CPU        |       2 |       1 |         tg128 |         25.31 ± 0.07 |

build: 5eb47b72 (3957)
```

In intel core i5-12600K, about 23-25 token/s

