---
title: SmolLM - blazingly fast and remarkably powerful
thumbnail: /blog/assets/smollm/banner.png
authors:
- user: loubnabnl
- user: anton-l
- user: eliebak
---

# SmolLM - blazingly fast and remarkably powerful

## TL;DR

This blog post introduces [SmolLM](https://huggingface.co/collections/HuggingFaceTB/smollm-models-6695016cad7167254ce15966), a family of state-of-the-art small models with 135M, 360M, and 1.7B parameters, trained on a new high-quality dataset.  It covers data curation, model evaluation, and usage.

## Introduction

There is increasing interest in small language models that can operate on local devices. This trend involves techniques such as distillation or quantization to compress large models, as well as training small models from scratch on large datasets. These approaches enable novel applications while dramatically reducing inference costs and improving user privacy. 

Microsoft's Phi series, Alibaba's Qwen2 (less than 2B), and Meta's MobileLLM demonstrate that small models can achieve impressive results when designed and trained thoughtfully. However, most of the details about the data curation and training of these models are not publicly available. 

In this blog post, we're excited to introduce [SmolLM](https://huggingface.co/collections/HuggingFaceTB/smollm-models-6695016cad7167254ce15966), a series of state-of-the-art small language models available in three sizes: 135M, 360M, and 1.7B parameters. These models are built on a meticulously curated high-quality training corpus, which we are releasing as [SmolLM-Corpus](https://huggingface.co/datasets/HuggingFaceTB/smollm-corpus). Smollm Corpus includes:

- **Cosmopedia v2**: A collection of synthetic textbooks and stories generated by Mixtral (28B tokens)
- **Python-Edu**: educational Python samples from The Stack (4B tokens)
- **FineWeb-Edu (deduplicated)**: educational web samples from FineWeb (220B tokens)

Our evaluations demonstrate that SmolLM models outperform other models in their size categories across a diverse set of benchmarks, testing common sense reasoning and world knowledge. In this blog post, we will go over the curation of each subset in the training corpus and then discuss the training and evaluation of SmolLM models. 

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Evaluation of SmolLM models on different reasoning and common knowledge benchmarks.</em>
</p>

## Data curation

### From Cosmopedia v1 to v2

[Cosmopedia v2](https://huggingface.co/datasets/HuggingFaceTB/smollm-corpus) is an enhanced version of Cosmopedia, the largest synthetic dataset for pre-training, consisting of over 30 million textbooks, blog posts, and stories generated by Mixtral-8x7B-Instruct-v0.1. Most of the samples are generated by prompting the model to generate content on specific topics using a web page referred to as a "seed sample", as shown in Figure 1. We use web samples to increase diversity and expand the range of prompts. You can find more details in this [blog post](https://huggingface.co/blog/cosmopedia).

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%201.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Figure 1. Example of a Cosmopedia prompt.</em>
</p>

To improve the dataset in v2, we tried two strategies:

- Using more capable models with the same prompts
- Optimizing the prompts themselves

For the first strategy, we experimented with llama3-70B-Instruct, Mixtral-8x22B-Instruct-v0.1, and Qwen1.5-72B-Chat but found no significant improvements when training models on textbooks generated by these alternatives. Therefore, in the remainder of this section, we will focus on the second strategy: how we improved the prompts.

#### The search for better topics and seed samples

Each prompt has three main components: the topic, the seed sample, and the generation style, which specifies the intended audience and the type of content we want the model to generate.

To ensure consistent generations, we need seed samples that are closely related to the given topic. In Cosmopedia v1, we ran clustering on FineWeb samples to identify both the topics and the corresponding web samples, as shown in Figure 2. This approach has two main limitations:

1. The topic list reflects the web/FineWeb clusters, which, while comprehensive, may limit our control over the topics.
2. The web samples within each cluster are not further filtered, potentially including some low-quality samples.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%202.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Figure 2. FineWeb clusters.</em>
</p>

Instead of this unsupervised clustering approach, in v2 we started with a predefined list of 34,000 topics using the [BISAC book classification](https://www.bisg.org/complete-bisac-subject-headings-list), a standard used to categorize books by subject that is both comprehensive and educationally focused. We started with 5,000 topics belonging to 51 categories and asked Mixtral to generate subtopics for certain topics. Below is the final distribution of subtopics in each category:

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%203.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Figure 3. Distribution of topics per top categories used for the prompts.</em>
</p>

After defining the topics, we still needed to find web pages related to them. Just like using a search engine to find content on a specific topic, we implemented a search tool to retrieve the most relevant pages for each topic. We ran this tool using our BISAC categories and their subtopics as queries on the FineWeb CC-MAIN-2024-10 and CC-MAIN-2023-50 dumps, which together consist of over 520 million samples. For each query, we retrieved 1,000 pages, ensuring we retrieved only the most relevant content. The code for deploying and running the search tool is available [here](https://github.com/huggingface/cosmopedia/tree/main/fulltext_search).

As a result, we compiled 34 million web pages across 34,000 topics. The next step was to determine which generation style worked best.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%204.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Figure 4. Topics and their retrieved samples in the category “Medical”.</em>
</p>

#### Generation Style

To determine the most effective generation style, we conducted ablation studies by training 1.8B models on 8B tokens from different subsets of Cosmopedia v1. For newly generated data, we only generated 2B tokens and trained for 4 epochs to save time (it takes approximately 1000 GPU hours to generate 2B tokens with Mixtral). We used the same training and evaluation setup as [FineWeb ablation models.](https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1) We ran each experiment twice with two different seeds and averaged the scores between the two runs. 

We compared the performance of the following subsets of Cosmopedia v1:

- The web textbooks subset
- The stories subset
- The Stanford & OpenStax subset

We found that textbooks based on topics and seed samples from curated sources such as Stanford and OpenStax provided the best overall performance, leading to MMLU and ARC benchmarks compared to web-based textbooks. Stories seemed to help with common sense benchmarks. After implementing the new topics and seed sample retrieval methods in v2, we were able to match the performance of curated sources using web seeds, confirming the quality of the new prompts. 

Next, we explored which audience style worked best. We generated textbooks using the same web textbook prompts but targeted two different audiences: middle school students and college students. We found that models trained on textbooks aimed primarily at middle school students gave the best score on all benchmarks except MMLU. This can be explained by the fact that most of these test basic common sense and elementary to intermediate science knowledge, while MMLU contains some questions that require advanced knowledge and expertise.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%205.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Evaluation of textbooks for different audiences.</em>
</p>
<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%206.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Evaluation of textbooks for different audiences.</em>
</p>

For v2, we decided to generate 40% of the content for middle school students, 30% for college students and 30% as a mix of other audiences and styles including in subsets we borrow from Cosmopedia v1 such as stories and Stanford courses based textbooks. Additionally, we generated 1B code textbooks based on Python seed samples from AutoMathText dataset.

Ultimately, we produced 39 million synthetic documents consisting of 28B tokens of textbooks, stories, articles, and code, with a diverse range of audiences and over 34,000 topics.

### FineWeb-Edu

FineWeb-Edu is a dataset we released a few months ago with FineWeb’s [technical report.](https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1)  It consists of **1.3T tokens** of educational web pages filtered from 🍷 FineWeb dataset.

We developed an [**educational quality classifier**](https://huggingface.co/HuggingFaceFW/fineweb-edu-classifier) using annotations generated by Llama3-70B-Instruct. We then used this classifier to retain only the most educational web pages from FineWeb. FineWeb-Edu outperforms FineWeb on popular benchmarks and shows the power of classifiers trained on synthetic data.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%207.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Comparison of FineWeb-Edu to other open web datasets.</em>
</p>

In Smollm-Corpus we include 220B deduplicated tokens from FineWeb.

### Stack-Edu-Python

We applied the same idea of FineWeb-Edu to Code. We used Llama3 to annotate 500,000 python samples from The Stack dataset and used them to train an [educational classifier](https://huggingface.co/HuggingFaceTB/python-edu-scorer) using the same recipe as the FineWeb-Edu classifier. We then applied this classifier on Python subset of StarCoder models training corpus. From the 40B Python tokens available, we retained only the samples with a score of 4 or higher, resulting in a refined dataset of 4B tokens.

The plot below compares Python-Edu to the unfiltered Python code and to using a less strict threshold of 3. We can see that the model trained on Python-Edu converges more than 3 times faster than the model trained on unfiltered Python code, achieving 16% pass@1 after only 12B tokens. 

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%208.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Comparison of Python-Edu to unfiltered Python code.</em>
</p>

## Training

SmolLM models are available in three sizes and were trained on the data mixture below:

- 135M and 360M models, each trained on 600B tokens from [Smollm-Corpus](https://huggingface.co/datasets/HuggingFaceTB/smollm-corpus)
- 1.7B model, trained on 1T tokens from Smollm-Corpus

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%209.png" alt=""  style="width: 60%; height: auto;"><br>
<em>Training mixture of SmolLM models.</em>
</p>

### Hyperparameters choice

We used a trapezoidal learning rate scheduler with a cooldown phase equal to 20% of the total training time. It's important to note that the original experiments with this schedule were conducted at a smaller scale, and we've adapted it for our larger models.

For the architecture of our 135M and 360M parameter models, we adopted a design similar to [MobileLLM](https://arxiv.org/abs/2402.14905), incorporating Grouped-Query Attention (GQA) and prioritizing depth over width. The 1.7B parameter model uses a more traditional architecture. For all three models we use embedding tying and a context length of 2048 tokens. This context length can be further extended with some long context fine-tuning.

The detailed architecture specifications for each model size are as follows:

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2010.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Architecture details of SmolLM models.</em>
</p>

We used a tokenizer trained on the Smollm Corpus with a vocab size of 49152. 

### Experiments

One advantage of using the trapezoidal scheduler is that it can reduce the time needed to perform scaling law experiments, as shown in [Hägele et al.](https://arxiv.org/pdf/2405.18392). We illustrate this with a small scaling law study on our smallest model, SmolLM-125M. We observed that performance continues to improve with longer training, even beyond the Chinchilla optimal point. Therefore, we decided to the 1.7B model on 1 trillion tokens and the 135M and 360M models on 600B tokens, as the performance gains after 400B tokens begin to slow on some benchmarks for these smaller models.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2011.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Evaluation of 125M SmolLM models trained on different numbers of tokens.</em>
</p>

We experimented with adding instruct datasets and upsampling the curated Cosmopedia subsets during the cooldown phase, but found no significant improvements. This may be because the primary data mixture is already of high quality, limiting the impact of these changes.

To track our training progress, we evaluate our two smallest models every 2B token. The following plot shows their performance on several benchmarks:

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2012.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Intermediate evaluation of SmolLM-135M and SmolLM-360M on different benchmarks.</em>
</p>

## Evaluation

In this section, we evaluate the performance of SmolLM models across different parameter sizes and compare them with the best models in their respective categories. We evaluate on a diverse set of benchmarks testing common sense reasoning and world knowledge. We use the same evaluation setup for all models using this [setup](https://github.com/huggingface/cosmopedia/tree/main/evaluation) with `lighteval` library. For HumanEval, we use [bigcode-evaluation-harness](We use temperature 0.2, top-p 0.95 with 20 samples.) with We use temperature 0.2, top-p 0.95 with 20 samples. For MobileLLM, which isn’t publicly available, we use the numbers reported in the paper whenever possible.

We find that:

- SmolLM-135M outperforms the current best model with less than 200M parameters, MobileLM-125M, despite being trained on only 600B tokens compared to MobileLM's 1T tokens.
- SmolLM**-**360M outperforms all models with less than 500M parameters, despite having fewer parameters and being trained on less than a trillion tokens (600B) as opposed to MobileLM-350M and Qwen2-500M.
- SmolLM-1.7B outperforms all other models with less than 2B parameters, including Phi1.5 from Microsoft, MobileLM-1.5B, and Qwen2-1.5B.
- SmolLM-1.7B shows strong Python coding performance with 24 pass@1. We note that the evaluation scorefor Qwen2-1.5B is different from the 31.1 pass@1 reported by Qwen team. We use temperature 0.2, top-p 0.95 with 20 samples.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2014.png" alt=""  style="width: 90%; height: auto;"><br>
<em>Comparison of SmolLM models to other SLMs. We evaluate all models on the same setup, except for MobieLLM, which isn't publicly available.</em>
</p>
<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/image.png" alt=""  style="width: 50%; height: auto;"><br>
<em>Evaluation of SmolLM models on HumanEval.</em>
</p>

We also instruction tuned the models using publicly available permissive instruction datasets. We trained all three models for one epoch on the permissive subset of the [WebInstructSub dataset](https://huggingface.co/datasets/TIGER-Lab/WebInstructSub), combined with StarCoder2-Self-OSS-Instruct. Following this, we performed DPO (Direct Preference Optimization) for one epoch: using [HelpSteer](https://huggingface.co/datasets/nvidia/HelpSteer) for the 135M and 1.7B models, and [argilla/dpo-mix-7k](https://huggingface.co/datasets/argilla/dpo-mix-7k) for the 360M model. We followed the training parameters from the Zephyr-Gemma recipe in the [alignment handbook](https://github.com/huggingface/alignment-handbook/blob/main/recipes/zephyr-7b-gemma/README.md), but adjusted the SFT (Supervised Fine-Tuning) learning rate to 3e-4.

The table below shows the performance of SmolLM-Instruct and other models on the IFEval benchmark (Prompt Strict Accuracy). Qwen2-1.5B-Instruct model scores the highest with 29.94, SmolLM-Instruct models provide a good balance between model size and performance, using only publicly available permissive datasets.

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2016.png" alt=""  style="width: 60%; height: auto;"><br>
<em>Evaluation of SmolLM-Instruct models on IFEval.</em>
</p>

## How to run locally ?

Our models are designed to be small and can run locally on various hardware configurations. For reference, an iPhone 15 has 6GB of DRAM, while an iPhone 15 Pro has 8GB. These memory requirements make our models suitable for deployment on a wide range of devices, from smartphones to laptops. We benchmarked the memory footprint of our three model sizes:

<p align="center">
 <img src="https://huggingface.co/datasets/HuggingFaceTB/images/resolve/main/Untitled%2013.png" alt=""  style="width: 60%; height: auto;"><br>
<em>Memory footprint of SmolLM models.</em>
</p>

Along with the transformers checkpoints, we released ONNX checkpoints and plan to add a GGUF version compatible with `llama.cpp`. You can find WebGPU demos SmolLM-135M and Smol-LM360M at [https://huggingface.co/spaces/HuggingFaceTB/SmolLM-135M-Instruct-WebGPU](https://huggingface.co/spaces/HuggingFaceTB/SmolLM-135M-Instruct-WebGPU) and [https://huggingface.co/spaces/HuggingFaceTB/SmolLM-360M-Instruct-WebGPU](https://huggingface.co/spaces/HuggingFaceTB/SmolLM-360M-Instruct-WebGPU).

## Conclusion

In this blog post we introduced SmolLM models, a new state-of-the-art family of small LLMs. They demonstrate that small language models can achieve high performance with efficient training on high-quality datasets, providing a strong balance between size and performance.

## Resources
- SmolLM models collection: [https://huggingface.co/collections/HuggingFaceTB/smollm-models-6695016cad7167254ce15966](https://huggingface.co/collections/HuggingFaceTB/smollm-models-6695016cad7167254ce15966)
- SmolLM-Corpus dataset: [https://huggingface.co/datasets/HuggingFaceTB/smollm-corpus](https://huggingface.co/datasets/HuggingFaceTB/smollm-corpus)
- WebGPU demo: [https://huggingface.co/spaces/HuggingFaceTB/SmolLM-135M-Instruct-WebGPU](https://huggingface.co/spaces/HuggingFaceTB/SmolLM-135M-Instruct-WebGPU) and [https://huggingface.co/spaces/HuggingFaceTB/SmolLM-360M-Instruct-WebGPU](https://huggingface.co/spaces/HuggingFaceTB/SmolLM-360M-Instruct-WebGPU)
