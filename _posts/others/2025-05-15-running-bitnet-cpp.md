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
```
# (Recommended) Create a new conda environment
conda create -n bitnet-cpp python=3.9
conda activate bitnet-cpp

pip install -r requirements.txt
```

After install dependency you should add below code to 3rdparty/llama.cpp/common/common.h and 3rdparty/llama.cpp/common/log.h because of bug in llama.cpp

```cpp
#include <chrono>
```

3. Build the project
```
# Manually download the model and run with local path
huggingface-cli download microsoft/BitNet-b1.58-2B-4T-gguf --local-dir models/BitNet-b1.58-2B-4T
python setup_env.py -md models/BitNet-b1.58-2B-4T -q i2_s

```

# basic usage
below code is run bitnet chat mode
```
# Run inference with the quantized model
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

>
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

In intell core i5-12600K, about 23-25 token/s

# Appendix
## logs with run_inference.py
```
(bitnet-cpp) D:\bitnet\BitNet>python run_inference.py -m models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf -p "You are a helpful assistant" -cnv -n 1000
warning: not compiled with GPU offload support, --gpu-layers option will be ignored
warning: see main README.md for information on enabling GPU BLAS support
build: 3957 (5eb47b72) with Clang 19.1.5 for x64
main: llama backend init
main: load the model and apply lora adapter, if any
llama_model_loader: loaded meta data with 24 key-value pairs and 332 tensors from models/BitNet-b1.58-2B-4T/ggml-model-i2_s.gguf (version GGUF V3 (latest))
llama_model_loader: Dumping metadata keys/values. Note: KV overrides do not apply in this output.
llama_model_loader: - kv   0:                       general.architecture str              = bitnet-b1.58
llama_model_loader: - kv   1:                               general.name str              = bitnet2b
llama_model_loader: - kv   2:                    bitnet-b1.58.vocab_size u32              = 128256
llama_model_loader: - kv   3:                bitnet-b1.58.context_length u32              = 4096
llama_model_loader: - kv   4:              bitnet-b1.58.embedding_length u32              = 2560
llama_model_loader: - kv   5:                   bitnet-b1.58.block_count u32              = 30
llama_model_loader: - kv   6:           bitnet-b1.58.feed_forward_length u32              = 6912
llama_model_loader: - kv   7:          bitnet-b1.58.rope.dimension_count u32              = 128
llama_model_loader: - kv   8:          bitnet-b1.58.attention.head_count u32              = 20
llama_model_loader: - kv   9:       bitnet-b1.58.attention.head_count_kv u32              = 5
llama_model_loader: - kv  10:               tokenizer.ggml.add_bos_token bool             = true
llama_model_loader: - kv  11: bitnet-b1.58.attention.layer_norm_rms_epsilon f32              = 0.000010
llama_model_loader: - kv  12:                bitnet-b1.58.rope.freq_base f32              = 500000.000000
llama_model_loader: - kv  13:                          general.file_type u32              = 40
llama_model_loader: - kv  14:                       tokenizer.ggml.model str              = gpt2
llama_model_loader: - kv  15:                      tokenizer.ggml.tokens arr[str,128256]  = ["!", "\"", "#", "$", "%", "&", "'", ...
llama_model_loader: - kv  16:                      tokenizer.ggml.scores arr[f32,128256]  = [0.000000, 0.000000, 0.000000, 0.0000...
llama_model_loader: - kv  17:                  tokenizer.ggml.token_type arr[i32,128256]  = [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, ...
llama_model_loader: - kv  18:                      tokenizer.ggml.merges arr[str,280147]  = ["Ġ Ġ", "Ġ ĠĠĠ", "ĠĠ ĠĠ", "...
llama_model_loader: - kv  19:                tokenizer.ggml.bos_token_id u32              = 128000
llama_model_loader: - kv  20:                tokenizer.ggml.eos_token_id u32              = 128001
llama_model_loader: - kv  21:            tokenizer.ggml.padding_token_id u32              = 128001
llama_model_loader: - kv  22:                    tokenizer.chat_template str              = {% for message in messages %}{% if lo...
llama_model_loader: - kv  23:               general.quantization_version u32              = 2
llama_model_loader: - type  f32:  121 tensors
llama_model_loader: - type  f16:    1 tensors
llama_model_loader: - type i2_s:  210 tensors
llm_load_vocab: missing pre-tokenizer type, using: 'default'
llm_load_vocab:
llm_load_vocab: ************************************
llm_load_vocab: GENERATION QUALITY WILL BE DEGRADED!
llm_load_vocab: CONSIDER REGENERATING THE MODEL
llm_load_vocab: ************************************
llm_load_vocab:
llm_load_vocab: control token: 128095 '<|reserved_special_token_90|>' is not marked as EOG
llm_load_vocab: control token: 128188 '<|reserved_special_token_183|>' is not marked as EOG
llm_load_vocab: control token: 128127 '<|reserved_special_token_122|>' is not marked as EOG
llm_load_vocab: control token: 128116 '<|reserved_special_token_111|>' is not marked as EOG
llm_load_vocab: control token: 128133 '<|reserved_special_token_128|>' is not marked as EOG
llm_load_vocab: control token: 128152 '<|reserved_special_token_147|>' is not marked as EOG
llm_load_vocab: control token: 128193 '<|reserved_special_token_188|>' is not marked as EOG
llm_load_vocab: control token: 128098 '<|reserved_special_token_93|>' is not marked as EOG
llm_load_vocab: control token: 128135 '<|reserved_special_token_130|>' is not marked as EOG
llm_load_vocab: control token: 128178 '<|reserved_special_token_173|>' is not marked as EOG
llm_load_vocab: control token: 128031 '<|reserved_special_token_26|>' is not marked as EOG
llm_load_vocab: control token: 128206 '<|reserved_special_token_201|>' is not marked as EOG
llm_load_vocab: control token: 128028 '<|reserved_special_token_23|>' is not marked as EOG
llm_load_vocab: control token: 128047 '<|reserved_special_token_42|>' is not marked as EOG
llm_load_vocab: control token: 128241 '<|reserved_special_token_236|>' is not marked as EOG
llm_load_vocab: control token: 128145 '<|reserved_special_token_140|>' is not marked as EOG
llm_load_vocab: control token: 128195 '<|reserved_special_token_190|>' is not marked as EOG
llm_load_vocab: control token: 128226 '<|reserved_special_token_221|>' is not marked as EOG
llm_load_vocab: control token: 128162 '<|reserved_special_token_157|>' is not marked as EOG
llm_load_vocab: control token: 128243 '<|reserved_special_token_238|>' is not marked as EOG
llm_load_vocab: control token: 128014 '<|reserved_special_token_9|>' is not marked as EOG
llm_load_vocab: control token: 128213 '<|reserved_special_token_208|>' is not marked as EOG
llm_load_vocab: control token: 128158 '<|reserved_special_token_153|>' is not marked as EOG
llm_load_vocab: control token: 128221 '<|reserved_special_token_216|>' is not marked as EOG
llm_load_vocab: control token: 128079 '<|reserved_special_token_74|>' is not marked as EOG
llm_load_vocab: control token: 128246 '<|reserved_special_token_241|>' is not marked as EOG
llm_load_vocab: control token: 128104 '<|reserved_special_token_99|>' is not marked as EOG
llm_load_vocab: control token: 128076 '<|reserved_special_token_71|>' is not marked as EOG
llm_load_vocab: control token: 128222 '<|reserved_special_token_217|>' is not marked as EOG
llm_load_vocab: control token: 128172 '<|reserved_special_token_167|>' is not marked as EOG
llm_load_vocab: control token: 128220 '<|reserved_special_token_215|>' is not marked as EOG
llm_load_vocab: control token: 128179 '<|reserved_special_token_174|>' is not marked as EOG
llm_load_vocab: control token: 128065 '<|reserved_special_token_60|>' is not marked as EOG
llm_load_vocab: control token: 128249 '<|reserved_special_token_244|>' is not marked as EOG
llm_load_vocab: control token: 128175 '<|reserved_special_token_170|>' is not marked as EOG
llm_load_vocab: control token: 128218 '<|reserved_special_token_213|>' is not marked as EOG
llm_load_vocab: control token: 128049 '<|reserved_special_token_44|>' is not marked as EOG
llm_load_vocab: control token: 128119 '<|reserved_special_token_114|>' is not marked as EOG
llm_load_vocab: control token: 128148 '<|reserved_special_token_143|>' is not marked as EOG
llm_load_vocab: control token: 128118 '<|reserved_special_token_113|>' is not marked as EOG
llm_load_vocab: control token: 128155 '<|reserved_special_token_150|>' is not marked as EOG
llm_load_vocab: control token: 128093 '<|reserved_special_token_88|>' is not marked as EOG
llm_load_vocab: control token: 128087 '<|reserved_special_token_82|>' is not marked as EOG
llm_load_vocab: control token: 128235 '<|reserved_special_token_230|>' is not marked as EOG
llm_load_vocab: control token: 128136 '<|reserved_special_token_131|>' is not marked as EOG
llm_load_vocab: control token: 128173 '<|reserved_special_token_168|>' is not marked as EOG
llm_load_vocab: control token: 128074 '<|reserved_special_token_69|>' is not marked as EOG
llm_load_vocab: control token: 128211 '<|reserved_special_token_206|>' is not marked as EOG
llm_load_vocab: control token: 128168 '<|reserved_special_token_163|>' is not marked as EOG
llm_load_vocab: control token: 128109 '<|reserved_special_token_104|>' is not marked as EOG
llm_load_vocab: control token: 128177 '<|reserved_special_token_172|>' is not marked as EOG
llm_load_vocab: control token: 128057 '<|reserved_special_token_52|>' is not marked as EOG
llm_load_vocab: control token: 128000 '<|begin_of_text|>' is not marked as EOG
llm_load_vocab: control token: 128149 '<|reserved_special_token_144|>' is not marked as EOG
llm_load_vocab: control token: 128113 '<|reserved_special_token_108|>' is not marked as EOG
llm_load_vocab: control token: 128069 '<|reserved_special_token_64|>' is not marked as EOG
llm_load_vocab: control token: 128056 '<|reserved_special_token_51|>' is not marked as EOG
llm_load_vocab: control token: 128091 '<|reserved_special_token_86|>' is not marked as EOG
llm_load_vocab: control token: 128184 '<|reserved_special_token_179|>' is not marked as EOG
llm_load_vocab: control token: 128100 '<|reserved_special_token_95|>' is not marked as EOG
llm_load_vocab: control token: 128124 '<|reserved_special_token_119|>' is not marked as EOG
llm_load_vocab: control token: 128020 '<|reserved_special_token_15|>' is not marked as EOG
llm_load_vocab: control token: 128034 '<|reserved_special_token_29|>' is not marked as EOG
llm_load_vocab: control token: 128225 '<|reserved_special_token_220|>' is not marked as EOG
llm_load_vocab: control token: 128002 '<|reserved_special_token_0|>' is not marked as EOG
llm_load_vocab: control token: 128006 '<|start_header_id|>' is not marked as EOG
llm_load_vocab: control token: 128088 '<|reserved_special_token_83|>' is not marked as EOG
llm_load_vocab: control token: 128041 '<|reserved_special_token_36|>' is not marked as EOG
llm_load_vocab: control token: 128215 '<|reserved_special_token_210|>' is not marked as EOG
llm_load_vocab: control token: 128208 '<|reserved_special_token_203|>' is not marked as EOG
llm_load_vocab: control token: 128070 '<|reserved_special_token_65|>' is not marked as EOG
llm_load_vocab: control token: 128165 '<|reserved_special_token_160|>' is not marked as EOG
llm_load_vocab: control token: 128180 '<|reserved_special_token_175|>' is not marked as EOG
llm_load_vocab: control token: 128231 '<|reserved_special_token_226|>' is not marked as EOG
llm_load_vocab: control token: 128232 '<|reserved_special_token_227|>' is not marked as EOG
llm_load_vocab: control token: 128064 '<|reserved_special_token_59|>' is not marked as EOG
llm_load_vocab: control token: 128036 '<|reserved_special_token_31|>' is not marked as EOG
llm_load_vocab: control token: 128103 '<|reserved_special_token_98|>' is not marked as EOG
llm_load_vocab: control token: 128247 '<|reserved_special_token_242|>' is not marked as EOG
llm_load_vocab: control token: 128170 '<|reserved_special_token_165|>' is not marked as EOG
llm_load_vocab: control token: 128123 '<|reserved_special_token_118|>' is not marked as EOG
llm_load_vocab: control token: 128044 '<|reserved_special_token_39|>' is not marked as EOG
llm_load_vocab: control token: 128237 '<|reserved_special_token_232|>' is not marked as EOG
llm_load_vocab: control token: 128042 '<|reserved_special_token_37|>' is not marked as EOG
llm_load_vocab: control token: 128192 '<|reserved_special_token_187|>' is not marked as EOG
llm_load_vocab: control token: 128075 '<|reserved_special_token_70|>' is not marked as EOG
llm_load_vocab: control token: 128134 '<|reserved_special_token_129|>' is not marked as EOG
llm_load_vocab: control token: 128183 '<|reserved_special_token_178|>' is not marked as EOG
llm_load_vocab: control token: 128045 '<|reserved_special_token_40|>' is not marked as EOG
llm_load_vocab: control token: 128073 '<|reserved_special_token_68|>' is not marked as EOG
llm_load_vocab: control token: 128026 '<|reserved_special_token_21|>' is not marked as EOG
llm_load_vocab: control token: 128010 '<|reserved_special_token_5|>' is not marked as EOG
llm_load_vocab: control token: 128194 '<|reserved_special_token_189|>' is not marked as EOG
llm_load_vocab: control token: 128053 '<|reserved_special_token_48|>' is not marked as EOG
llm_load_vocab: control token: 128120 '<|reserved_special_token_115|>' is not marked as EOG
llm_load_vocab: control token: 128092 '<|reserved_special_token_87|>' is not marked as EOG
llm_load_vocab: control token: 128086 '<|reserved_special_token_81|>' is not marked as EOG
llm_load_vocab: control token: 128054 '<|reserved_special_token_49|>' is not marked as EOG
llm_load_vocab: control token: 128160 '<|reserved_special_token_155|>' is not marked as EOG
llm_load_vocab: control token: 128005 '<|reserved_special_token_3|>' is not marked as EOG
llm_load_vocab: control token: 128050 '<|reserved_special_token_45|>' is not marked as EOG
llm_load_vocab: control token: 128157 '<|reserved_special_token_152|>' is not marked as EOG
llm_load_vocab: control token: 128219 '<|reserved_special_token_214|>' is not marked as EOG
llm_load_vocab: control token: 128032 '<|reserved_special_token_27|>' is not marked as EOG
llm_load_vocab: control token: 128159 '<|reserved_special_token_154|>' is not marked as EOG
llm_load_vocab: control token: 128202 '<|reserved_special_token_197|>' is not marked as EOG
llm_load_vocab: control token: 128106 '<|reserved_special_token_101|>' is not marked as EOG
llm_load_vocab: control token: 128182 '<|reserved_special_token_177|>' is not marked as EOG
llm_load_vocab: control token: 128111 '<|reserved_special_token_106|>' is not marked as EOG
llm_load_vocab: control token: 128156 '<|reserved_special_token_151|>' is not marked as EOG
llm_load_vocab: control token: 128176 '<|reserved_special_token_171|>' is not marked as EOG
llm_load_vocab: control token: 128112 '<|reserved_special_token_107|>' is not marked as EOG
llm_load_vocab: control token: 128084 '<|reserved_special_token_79|>' is not marked as EOG
llm_load_vocab: control token: 128110 '<|reserved_special_token_105|>' is not marked as EOG
llm_load_vocab: control token: 128051 '<|reserved_special_token_46|>' is not marked as EOG
llm_load_vocab: control token: 128027 '<|reserved_special_token_22|>' is not marked as EOG
llm_load_vocab: control token: 128167 '<|reserved_special_token_162|>' is not marked as EOG
llm_load_vocab: control token: 128008 '<|reserved_special_token_4|>' is not marked as EOG
llm_load_vocab: control token: 128061 '<|reserved_special_token_56|>' is not marked as EOG
llm_load_vocab: control token: 128115 '<|reserved_special_token_110|>' is not marked as EOG
llm_load_vocab: control token: 128203 '<|reserved_special_token_198|>' is not marked as EOG
llm_load_vocab: control token: 128096 '<|reserved_special_token_91|>' is not marked as EOG
llm_load_vocab: control token: 128130 '<|reserved_special_token_125|>' is not marked as EOG
llm_load_vocab: control token: 128187 '<|reserved_special_token_182|>' is not marked as EOG
llm_load_vocab: control token: 128094 '<|reserved_special_token_89|>' is not marked as EOG
llm_load_vocab: control token: 128083 '<|reserved_special_token_78|>' is not marked as EOG
llm_load_vocab: control token: 128117 '<|reserved_special_token_112|>' is not marked as EOG
llm_load_vocab: control token: 128190 '<|reserved_special_token_185|>' is not marked as EOG
llm_load_vocab: control token: 128046 '<|reserved_special_token_41|>' is not marked as EOG
llm_load_vocab: control token: 128239 '<|reserved_special_token_234|>' is not marked as EOG
llm_load_vocab: control token: 128139 '<|reserved_special_token_134|>' is not marked as EOG
llm_load_vocab: control token: 128185 '<|reserved_special_token_180|>' is not marked as EOG
llm_load_vocab: control token: 128141 '<|reserved_special_token_136|>' is not marked as EOG
llm_load_vocab: control token: 128244 '<|reserved_special_token_239|>' is not marked as EOG
llm_load_vocab: control token: 128062 '<|reserved_special_token_57|>' is not marked as EOG
llm_load_vocab: control token: 128114 '<|reserved_special_token_109|>' is not marked as EOG
llm_load_vocab: control token: 128030 '<|reserved_special_token_25|>' is not marked as EOG
llm_load_vocab: control token: 128181 '<|reserved_special_token_176|>' is not marked as EOG
llm_load_vocab: control token: 128037 '<|reserved_special_token_32|>' is not marked as EOG
llm_load_vocab: control token: 128201 '<|reserved_special_token_196|>' is not marked as EOG
llm_load_vocab: control token: 128207 '<|reserved_special_token_202|>' is not marked as EOG
llm_load_vocab: control token: 128242 '<|reserved_special_token_237|>' is not marked as EOG
llm_load_vocab: control token: 128132 '<|reserved_special_token_127|>' is not marked as EOG
llm_load_vocab: control token: 128068 '<|reserved_special_token_63|>' is not marked as EOG
llm_load_vocab: control token: 128150 '<|reserved_special_token_145|>' is not marked as EOG
llm_load_vocab: control token: 128191 '<|reserved_special_token_186|>' is not marked as EOG
llm_load_vocab: control token: 128174 '<|reserved_special_token_169|>' is not marked as EOG
llm_load_vocab: control token: 128233 '<|reserved_special_token_228|>' is not marked as EOG
llm_load_vocab: control token: 128245 '<|reserved_special_token_240|>' is not marked as EOG
llm_load_vocab: control token: 128238 '<|reserved_special_token_233|>' is not marked as EOG
llm_load_vocab: control token: 128209 '<|reserved_special_token_204|>' is not marked as EOG
llm_load_vocab: control token: 128204 '<|reserved_special_token_199|>' is not marked as EOG
llm_load_vocab: control token: 128001 '<|end_of_text|>' is not marked as EOG
llm_load_vocab: control token: 128003 '<|reserved_special_token_1|>' is not marked as EOG
llm_load_vocab: control token: 128004 '<|reserved_special_token_2|>' is not marked as EOG
llm_load_vocab: control token: 128007 '<|end_header_id|>' is not marked as EOG
llm_load_vocab: control token: 128011 '<|reserved_special_token_6|>' is not marked as EOG
llm_load_vocab: control token: 128012 '<|reserved_special_token_7|>' is not marked as EOG
llm_load_vocab: control token: 128013 '<|reserved_special_token_8|>' is not marked as EOG
llm_load_vocab: control token: 128015 '<|reserved_special_token_10|>' is not marked as EOG
llm_load_vocab: control token: 128016 '<|reserved_special_token_11|>' is not marked as EOG
llm_load_vocab: control token: 128017 '<|reserved_special_token_12|>' is not marked as EOG
llm_load_vocab: control token: 128018 '<|reserved_special_token_13|>' is not marked as EOG
llm_load_vocab: control token: 128019 '<|reserved_special_token_14|>' is not marked as EOG
llm_load_vocab: control token: 128021 '<|reserved_special_token_16|>' is not marked as EOG
llm_load_vocab: control token: 128022 '<|reserved_special_token_17|>' is not marked as EOG
llm_load_vocab: control token: 128023 '<|reserved_special_token_18|>' is not marked as EOG
llm_load_vocab: control token: 128024 '<|reserved_special_token_19|>' is not marked as EOG
llm_load_vocab: control token: 128025 '<|reserved_special_token_20|>' is not marked as EOG
llm_load_vocab: control token: 128029 '<|reserved_special_token_24|>' is not marked as EOG
llm_load_vocab: control token: 128033 '<|reserved_special_token_28|>' is not marked as EOG
llm_load_vocab: control token: 128035 '<|reserved_special_token_30|>' is not marked as EOG
llm_load_vocab: control token: 128038 '<|reserved_special_token_33|>' is not marked as EOG
llm_load_vocab: control token: 128039 '<|reserved_special_token_34|>' is not marked as EOG
llm_load_vocab: control token: 128040 '<|reserved_special_token_35|>' is not marked as EOG
llm_load_vocab: control token: 128043 '<|reserved_special_token_38|>' is not marked as EOG
llm_load_vocab: control token: 128048 '<|reserved_special_token_43|>' is not marked as EOG
llm_load_vocab: control token: 128052 '<|reserved_special_token_47|>' is not marked as EOG
llm_load_vocab: control token: 128055 '<|reserved_special_token_50|>' is not marked as EOG
llm_load_vocab: control token: 128058 '<|reserved_special_token_53|>' is not marked as EOG
llm_load_vocab: control token: 128059 '<|reserved_special_token_54|>' is not marked as EOG
llm_load_vocab: control token: 128060 '<|reserved_special_token_55|>' is not marked as EOG
llm_load_vocab: control token: 128063 '<|reserved_special_token_58|>' is not marked as EOG
llm_load_vocab: control token: 128066 '<|reserved_special_token_61|>' is not marked as EOG
llm_load_vocab: control token: 128067 '<|reserved_special_token_62|>' is not marked as EOG
llm_load_vocab: control token: 128071 '<|reserved_special_token_66|>' is not marked as EOG
llm_load_vocab: control token: 128072 '<|reserved_special_token_67|>' is not marked as EOG
llm_load_vocab: control token: 128077 '<|reserved_special_token_72|>' is not marked as EOG
llm_load_vocab: control token: 128078 '<|reserved_special_token_73|>' is not marked as EOG
llm_load_vocab: control token: 128080 '<|reserved_special_token_75|>' is not marked as EOG
llm_load_vocab: control token: 128081 '<|reserved_special_token_76|>' is not marked as EOG
llm_load_vocab: control token: 128082 '<|reserved_special_token_77|>' is not marked as EOG
llm_load_vocab: control token: 128085 '<|reserved_special_token_80|>' is not marked as EOG
llm_load_vocab: control token: 128089 '<|reserved_special_token_84|>' is not marked as EOG
llm_load_vocab: control token: 128090 '<|reserved_special_token_85|>' is not marked as EOG
llm_load_vocab: control token: 128097 '<|reserved_special_token_92|>' is not marked as EOG
llm_load_vocab: control token: 128099 '<|reserved_special_token_94|>' is not marked as EOG
llm_load_vocab: control token: 128101 '<|reserved_special_token_96|>' is not marked as EOG
llm_load_vocab: control token: 128102 '<|reserved_special_token_97|>' is not marked as EOG
llm_load_vocab: control token: 128105 '<|reserved_special_token_100|>' is not marked as EOG
llm_load_vocab: control token: 128107 '<|reserved_special_token_102|>' is not marked as EOG
llm_load_vocab: control token: 128108 '<|reserved_special_token_103|>' is not marked as EOG
llm_load_vocab: control token: 128121 '<|reserved_special_token_116|>' is not marked as EOG
llm_load_vocab: control token: 128122 '<|reserved_special_token_117|>' is not marked as EOG
llm_load_vocab: control token: 128125 '<|reserved_special_token_120|>' is not marked as EOG
llm_load_vocab: control token: 128126 '<|reserved_special_token_121|>' is not marked as EOG
llm_load_vocab: control token: 128128 '<|reserved_special_token_123|>' is not marked as EOG
llm_load_vocab: control token: 128129 '<|reserved_special_token_124|>' is not marked as EOG
llm_load_vocab: control token: 128131 '<|reserved_special_token_126|>' is not marked as EOG
llm_load_vocab: control token: 128137 '<|reserved_special_token_132|>' is not marked as EOG
llm_load_vocab: control token: 128138 '<|reserved_special_token_133|>' is not marked as EOG
llm_load_vocab: control token: 128140 '<|reserved_special_token_135|>' is not marked as EOG
llm_load_vocab: control token: 128142 '<|reserved_special_token_137|>' is not marked as EOG
llm_load_vocab: control token: 128143 '<|reserved_special_token_138|>' is not marked as EOG
llm_load_vocab: control token: 128144 '<|reserved_special_token_139|>' is not marked as EOG
llm_load_vocab: control token: 128146 '<|reserved_special_token_141|>' is not marked as EOG
llm_load_vocab: control token: 128147 '<|reserved_special_token_142|>' is not marked as EOG
llm_load_vocab: control token: 128151 '<|reserved_special_token_146|>' is not marked as EOG
llm_load_vocab: control token: 128153 '<|reserved_special_token_148|>' is not marked as EOG
llm_load_vocab: control token: 128154 '<|reserved_special_token_149|>' is not marked as EOG
llm_load_vocab: control token: 128161 '<|reserved_special_token_156|>' is not marked as EOG
llm_load_vocab: control token: 128163 '<|reserved_special_token_158|>' is not marked as EOG
llm_load_vocab: control token: 128164 '<|reserved_special_token_159|>' is not marked as EOG
llm_load_vocab: control token: 128166 '<|reserved_special_token_161|>' is not marked as EOG
llm_load_vocab: control token: 128169 '<|reserved_special_token_164|>' is not marked as EOG
llm_load_vocab: control token: 128171 '<|reserved_special_token_166|>' is not marked as EOG
llm_load_vocab: control token: 128186 '<|reserved_special_token_181|>' is not marked as EOG
llm_load_vocab: control token: 128189 '<|reserved_special_token_184|>' is not marked as EOG
llm_load_vocab: control token: 128196 '<|reserved_special_token_191|>' is not marked as EOG
llm_load_vocab: control token: 128197 '<|reserved_special_token_192|>' is not marked as EOG
llm_load_vocab: control token: 128198 '<|reserved_special_token_193|>' is not marked as EOG
llm_load_vocab: control token: 128199 '<|reserved_special_token_194|>' is not marked as EOG
llm_load_vocab: control token: 128200 '<|reserved_special_token_195|>' is not marked as EOG
llm_load_vocab: control token: 128205 '<|reserved_special_token_200|>' is not marked as EOG
llm_load_vocab: control token: 128210 '<|reserved_special_token_205|>' is not marked as EOG
llm_load_vocab: control token: 128212 '<|reserved_special_token_207|>' is not marked as EOG
llm_load_vocab: control token: 128214 '<|reserved_special_token_209|>' is not marked as EOG
llm_load_vocab: control token: 128216 '<|reserved_special_token_211|>' is not marked as EOG
llm_load_vocab: control token: 128217 '<|reserved_special_token_212|>' is not marked as EOG
llm_load_vocab: control token: 128223 '<|reserved_special_token_218|>' is not marked as EOG
llm_load_vocab: control token: 128224 '<|reserved_special_token_219|>' is not marked as EOG
llm_load_vocab: control token: 128227 '<|reserved_special_token_222|>' is not marked as EOG
llm_load_vocab: control token: 128228 '<|reserved_special_token_223|>' is not marked as EOG
llm_load_vocab: control token: 128229 '<|reserved_special_token_224|>' is not marked as EOG
llm_load_vocab: control token: 128230 '<|reserved_special_token_225|>' is not marked as EOG
llm_load_vocab: control token: 128234 '<|reserved_special_token_229|>' is not marked as EOG
llm_load_vocab: control token: 128236 '<|reserved_special_token_231|>' is not marked as EOG
llm_load_vocab: control token: 128240 '<|reserved_special_token_235|>' is not marked as EOG
llm_load_vocab: control token: 128248 '<|reserved_special_token_243|>' is not marked as EOG
llm_load_vocab: control token: 128250 '<|reserved_special_token_245|>' is not marked as EOG
llm_load_vocab: control token: 128251 '<|reserved_special_token_246|>' is not marked as EOG
llm_load_vocab: control token: 128252 '<|reserved_special_token_247|>' is not marked as EOG
llm_load_vocab: control token: 128253 '<|reserved_special_token_248|>' is not marked as EOG
llm_load_vocab: control token: 128254 '<|reserved_special_token_249|>' is not marked as EOG
llm_load_vocab: control token: 128255 '<|reserved_special_token_250|>' is not marked as EOG
llm_load_vocab: special_eos_id is not in special_eog_ids - the tokenizer config may be incorrect
llm_load_vocab: special tokens cache size = 256
llm_load_vocab: token to piece cache size = 0.8000 MB
llm_load_print_meta: format           = GGUF V3 (latest)
llm_load_print_meta: arch             = bitnet-b1.58
llm_load_print_meta: vocab type       = BPE
llm_load_print_meta: n_vocab          = 128256
llm_load_print_meta: n_merges         = 280147
llm_load_print_meta: vocab_only       = 0
llm_load_print_meta: n_ctx_train      = 4096
llm_load_print_meta: n_embd           = 2560
llm_load_print_meta: n_layer          = 30
llm_load_print_meta: n_head           = 20
llm_load_print_meta: n_head_kv        = 5
llm_load_print_meta: n_rot            = 128
llm_load_print_meta: n_swa            = 0
llm_load_print_meta: n_embd_head_k    = 128
llm_load_print_meta: n_embd_head_v    = 128
llm_load_print_meta: n_gqa            = 4
llm_load_print_meta: n_embd_k_gqa     = 640
llm_load_print_meta: n_embd_v_gqa     = 640
llm_load_print_meta: f_norm_eps       = 0.0e+00
llm_load_print_meta: f_norm_rms_eps   = 1.0e-05
llm_load_print_meta: f_clamp_kqv      = 0.0e+00
llm_load_print_meta: f_max_alibi_bias = 0.0e+00
llm_load_print_meta: f_logit_scale    = 0.0e+00
llm_load_print_meta: n_ff             = 6912
llm_load_print_meta: n_expert         = 0
llm_load_print_meta: n_expert_used    = 0
llm_load_print_meta: causal attn      = 1
llm_load_print_meta: pooling type     = 0
llm_load_print_meta: rope type        = 2
llm_load_print_meta: rope scaling     = linear
llm_load_print_meta: freq_base_train  = 500000.0
llm_load_print_meta: freq_scale_train = 1
llm_load_print_meta: n_ctx_orig_yarn  = 4096
llm_load_print_meta: rope_finetuned   = unknown
llm_load_print_meta: ssm_d_conv       = 0
llm_load_print_meta: ssm_d_inner      = 0
llm_load_print_meta: ssm_d_state      = 0
llm_load_print_meta: ssm_dt_rank      = 0
llm_load_print_meta: ssm_dt_b_c_rms   = 0
llm_load_print_meta: model type       = 2B
llm_load_print_meta: model ftype      = I2_S - 2 bpw ternary
llm_load_print_meta: model params     = 2.41 B
llm_load_print_meta: model size       = 1.10 GiB (3.91 BPW)
llm_load_print_meta: general.name     = bitnet2b
llm_load_print_meta: BOS token        = 128000 '<|begin_of_text|>'
llm_load_print_meta: EOS token        = 128001 '<|end_of_text|>'
llm_load_print_meta: EOT token        = 128009 '<|eot_id|>'
llm_load_print_meta: PAD token        = 128001 '<|end_of_text|>'
llm_load_print_meta: LF token         = 128 'Ä'
llm_load_print_meta: EOG token        = 128001 '<|end_of_text|>'
llm_load_print_meta: EOG token        = 128009 '<|eot_id|>'
llm_load_print_meta: max token length = 256
llm_load_tensors: ggml ctx size =    0.15 MiB
llm_load_tensors:        CPU buffer size =  1124.81 MiB
...............................
llama_new_context_with_model: n_batch is less than GGML_KQ_MASK_PAD - increasing to 32
llama_new_context_with_model: n_ctx      = 2048
llama_new_context_with_model: n_batch    = 32
llama_new_context_with_model: n_ubatch   = 32
llama_new_context_with_model: flash_attn = 0
llama_new_context_with_model: freq_base  = 500000.0
llama_new_context_with_model: freq_scale = 1
llama_kv_cache_init:        CPU KV buffer size =   150.00 MiB
llama_new_context_with_model: KV self size  =  150.00 MiB, K (f16):   75.00 MiB, V (f16):   75.00 MiB
llama_new_context_with_model:        CPU  output buffer size =     0.49 MiB
llama_new_context_with_model:        CPU compute buffer size =    15.97 MiB
llama_new_context_with_model: graph nodes  = 1116
llama_new_context_with_model: graph splits = 1
common_init_from_params: warming up the model with an empty run - please wait ... (--no-warmup to disable)
main: llama threadpool init, n_threads = 2
main: chat template example:
System: You are a helpful assistantUser: Hello<|eot_id|>Assistant: Hi thereUser: How are you?<|eot_id|>Assistant:

system_info: n_threads = 2 (n_threads_batch = 2) / 16 | AVX = 1 | AVX_VNNI = 0 | AVX2 = 1 | AVX512 = 0 | AVX512_VBMI = 0 | AVX512_VNNI = 0 | AVX512_BF16 = 0 | FMA = 1 | NEON = 0 | SVE = 0 | ARM_FMA = 0 | F16C = 1 | FP16_VA = 0 | RISCV_VECT = 0 | WASM_SIMD = 0 | BLAS = 0 | SSE3 = 1 | SSSE3 = 1 | VSX = 0 | MATMUL_INT8 = 0 | LLAMAFILE = 1 |

main: interactive mode on.
sampler seed: 2197981108
sampler params:
        repeat_last_n = 64, repeat_penalty = 1.000, frequency_penalty = 0.000, presence_penalty = 0.000
        top_k = 40, tfs_z = 1.000, top_p = 0.950, min_p = 0.050, typical_p = 1.000, temp = 0.800
        mirostat = 0, mirostat_lr = 0.100, mirostat_ent = 5.000
sampler chain: logits -> logit-bias -> penalties -> top-k -> tail-free -> typical -> top-p -> min-p -> temp-ext -> softmax -> dist
generate: n_ctx = 2048, n_batch = 1, n_predict = 1000, n_keep = 1

== Running in interactive mode. ==
 - Press Ctrl+C to interject at any time.
 - Press Return to return control to the AI.
 - To return control without starting a new line, end your input with '/'.
 - If you want to submit another line, end your input with '\'.

System: You are a helpful assistant
```