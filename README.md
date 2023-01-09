  
⚠️ **Pre release work in progress** ⚠️

# LLMOps for Large Language Model based applications 

All but most trivial LLM applications require complex prompt handling, development, evaluation, secure use, and deployment techniques. 

![bosquet](https://upload.wikimedia.org/wikipedia/commons/thumb/4/4f/42_Apollo_in_bosquet_F%C3%A4cher%2C_gardens_of_Sch%C3%B6nbrunn_03.jpg/640px-42_Apollo_in_bosquet_F%C3%A4cher%2C_gardens_of_Sch%C3%B6nbrunn_03.jpg)


Bosquet is building functionality to aid with those challenges:
* Support access to all main LLM models: [GPT](https://openai.com/api/), [Bloom](https://bigscience.huggingface.co/blog/bloom), and [Stable Diffusion](https://stability.ai/blog/stable-diffusion-v2-release) to start with.
* Provide scaffolding for prompt building methods: Role Promoting, Chain of Thought, Zero-Shot CoT, Self Consistency, and more.
* Vulnerability assessment and monitoring. How possible are prompt leak or injection attacks? Can prompt generate harmful content?
* Prompt quality evaluation.
* Developed ant tested prompt deployment to [Cloudflare Workers](https://workers.cloudflare.com/), [AWS Lambda](https://aws.amazon.com/lambda/), or self host via REST API.

## Prompt compilation

Bosquet allows defining prompts that depend on each other, their generated outputs, and supplied data. Let us take a simple example where we want to generate a *synopsis* of the play and a *review* of that play.

The play takes in two parameters: the *title* and *style*. The generation of synopsis depends on the text generated from the *play* prompt. 

The picture shows prompt templates, dependencies, and the sequence needed to produce the final result.

![prompt chaining](/doc/img/chained-generation.png)

1. First data needs to be filled in: *title* - "The Parade" and *style* - "horror"
1. These are all the dependencies needed for *synopsis* generation, and at the place specified with `((bosquet.openai/complete))` an OpenAI API is called to get the results.
1. Once *synopsis* is completed, *review* can be done. The *synopsis/completion* dependency is automatically fulfilled and the *review* prompt `((bosquet.openai/complete))` will be called to produce the review 
1. Generated text for reiview will go under *review/completion* key.

Full Clojure code to make the call first prompts, data, and OpenAI API configuration needs to be defined (config only shows few parameters, all are supported and have default values)

```clojure
(def prompt
  {:review
   "You are a play critic from the New York Times.
Given the synopsis of play, it is your job to write a review for that play.

Play Synopsis:
{{synopsis/completion}}
Review from a New York Times play critic of the above play:
((bosquet.openai/complete))"

   :synopsis
   "You are a playwright. Given the title of play and the desired style,
it is your job to write a synopsis for that title.

Title: {{title}}
Style: {{style}}
Playwright: This is a synopsis for the above play:
((bosquet.openai/complete))"})

(def config
  {:model "text-davinci-003"
   :temperature 0.85})

```

And the call to produce completions

```clojure
(bosquet.generator/complete prompt data config)

```

Bosquet will produce a map with the following keys:
1. *review/full-text* and *synopsis/full-text* containing the fully compiled prompt with all the dependencies fullfilled
1. *review/completion* and *synopsis/completion* containing only the text generated by OpenAI API call

## Prompt Templates

Bosquet uses [Selmer](https://github.com/yogthos/Selmer) to define its templates with all the functionality comming from Selmer's templating language:
* filters
* loops
* branches
* default values
to name a few.

A template example using for loop to fill in the data passed in as a collection

![selmer template](/doc/img/selmer-template.png)


## Prompt Palettes

Bosquet provides predefined prompting patterns. For example, the above-described Chain of Thought can be done with the:

``` clojure
(require '[bosquet.prompt-pattern :as pp])

(def roger-cot
  (pp/chain-of-though
    "Roger has 5 tennis balls. He buys 2 more cans of tennis balls. Each can has 3 tennis balls. How many tennis balls does he have now?"
    "Roger started with 5 balls. 2 cans of 3 tennis balls each is 6 tennis balls. 5 + 6 = 11."
    "The answer is 11."))

(roger-cot 
  "The cafeteria had 23 apples. If they used 20 to make lunch and bought 6 more, how many apples do they have?")
```

