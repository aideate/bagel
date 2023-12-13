# A bagel, with everything

The name of this project was shamelessly stolen from from Everything, Everywhere, All at Once.

![bagel](resources/bagel.webp)

## Data selection.

The first step in the process is creating a dataset.
In this case, we're actually creating a composite dataset, consisting of both supervised fine-tuning data (SFT) and direct preference optimization (DPO) data.

All instruction data, that is, data that is not plain text (like project Gutenberg and items from Cinematika) or DPO, is converted into ShareGPT format so it's easier to work with.

See the corresponding code in `bagel/data_sources/*.py` for full implementation for each data source.

Deduplication is done by creating a uuid v5 of the instruction/text, then only adding items not previously seen (where datasets are loaded in order of the confidence score I assign them).
This means that if an instruction is in data source "Foo" with confidence 4 as well as in data source "Bar" with confidence score 2, only the entry from "Foo" will be taken.

### SFT data sources

*Yes, you will see benchmark names in the list, but this only uses the train splits, and a decontamination by cosine similarity is performed at the end as a sanity check*

- [ai2_arc](https://huggingface.co/datasets/ai2_arc)
  - Abstraction and reasoning dataset, useful in measuring "intelligence" to a certain extent.
- [airoboros](https://huggingface.co/datasets/unalignment/spicy-3.1)
  - Variety of categories of synthetic instructions generated by gpt-4.
- [apps](https://huggingface.co/datasets/codeparrot/apps)
  - Python coding dataset with 10k problems.
- [belebele](https://huggingface.co/datasets/facebook/belebele)
  - Multi-lingual reading comprehension dataset.
- [boolq](https://huggingface.co/datasets/boolq)
  - Corpus of yes/no questions (which can be surprisingly difficult for AI to answer apparently?)
- [cinematika](https://huggingface.co/datasets/jondurbin/cinematika-v0.1) (instruction and plain text)
  - RP-style data synthesized from movie scripts so the model isn't quite as boring as it otherwise would be.
- [drop](https://huggingface.co/datasets/drop)
  - More reading comprehension.
- [gutenberg](https://www.gutenberg.org/) (plain text)
  - Books/plain text, again to make the model less boring, only a handful of examples supported by [chapterize](https://github.com/JonathanReeve/chapterize)
- [lmsys_chat_1m](https://huggingface.co/datasets/lmsys/lmsys-chat-1m) (only gpt-4 items, also used for DPO)
  - Chats collected by the lmsys chat arena, containing a wide variety of chats with various models.
- [mathinstruct](https://huggingface.co/datasets/TIGER-Lab/MathInstruct)
  - Composite dataset with a variety of math-related tasks and problem/question formats.
- [mmlu](https://huggingface.co/datasets/cais/mmlu)
  - Massive Multitask Language Understanding - a wide variety of questions about various subject matters.
- [natural_instructions](https://huggingface.co/datasets/Muennighoff/natural-instructions)
  - Millions of instructions from 1600+ task categories (sampled down substantially, stratified by task type)
- [openbookqa](https://huggingface.co/datasets/openbookqa)
  - Question answering dataset.
- [piqa](https://huggingface.co/datasets/piqa)
  - Phyiscal interaction question answering.
- [python_alpaca](https://huggingface.co/datasets/Vezora/Tested-22k-Python-Alpaca)
  - Python instruction response pairs, validated as functional.
- [rosetta_code](https://huggingface.co/datasets/cakiki/rosetta-code)
  - Code problems and solutions in a variety of programming languages taken from rosettacode.org.
- [slimorca](https://huggingface.co/datasets/Open-Orca/SlimOrca)
  - Collection of ~500k gpt-4 verified chats from OpenOrca.
- [spider](https://huggingface.co/datasets/spider)
  - SQL-targeted dataset.
- [squad_v2](https://huggingface.co/datasets/squad_v2)
  - Contextual question answering (RAG).
- [synthia](https://huggingface.co/datasets/migtissera/Synthia-v1.3)
  - GPT-4 generated data using advanced prompting from Migel Tissera.
- [winogrande](https://huggingface.co/datasets/winogrande)
  - Fill in the blank style prompts.

### DPO data sources
- [airoboros 3.1](https://huggingface.co/datasets/unalignment/spicy-3.1) vs [airoboros 2.2.1](https://huggingface.co/datasets/jondurbin/airoboros-gpt4-1.4.1)
  - The creative/writing tasks from airoboros-2.2.1 were re-generated using gpt4-0314 and a custom prompt to get longer, more creative, less clichè responses for airoboros 3.1, so we can use the shorter/boring version as the "rejected" value and the rerolled response as "chosen"
- [helpsteer](https://huggingface.co/datasets/nvidia/HelpSteer)
  - Really neat dataset provided by the folks at NVidia with human annotation across a variety of metrics.  Only items with the highest "correctness" value were used for DPO here, with the highest scoring output as "chosen" and random lower scoring value as "rejected"
- [orca_dpo_pairs](https://huggingface.co/datasets/Intel/orca_dpo_pairs)
  - Another interesting dataset by Intel, which provides various DPO pairs generated from prompts included in the SlimOrca dataset.
- [toxic-dpo](https://huggingface.co/datasets/unalignment/toxic-dpo-v0.1)
  - __*highly toxic and potentially illegal content!*__ De-censorship, for academic and lawful purposes only, of course.  Generated by llama-2-70b via prompt engineering.
- [truthy](https://huggingface.co/datasets/jondurbin/truthy-dpo-v0.1)
  - DPO pairs meant to increase truthfulness of the model, e.g. common misconceptions, differentiate between AI assistants and roleplayed human in terms of corporeal awareness/locality/etc.
- [ultrafeedback](https://huggingface.co/datasets/allenai/ultrafeedback_binarized_cleaned)
  - One of the bits of magic behind the Zephyr model.  Only the items with a chosen score of 8 or higher were included.

Only the train splits were used (if a split was provided), and an additional pass of decontamination is performed using approximate nearest neighbor search (via faiss).

### Total dataset size

The deduplicated and decontamined list of instructions contains 1,671,822 items:

- 1,602,217 SFT/instructions
- 59,247 DPO pairs
- 1606 with both SFT and DPO data

Keep in mind, this number becomes 4x larger when applying the various prompt formats.

## Prompt formatting

In sticking with the theme of the bagel, I didn't want to use a single prompt format, so I used 4 - vicuna, llama-2, alpaca, and chat-ml (sorta).
I also didn't want to randomly select a single prompt format for each item (hoping each instruction would generalize more when used in a variety of prompt formats), so each instruction is actually converted into every prompt format.

This means each epoch of our fine-tune is really basically 4 epochs.  So, for the fine-tunes, I would recommend only doing 1 epoch (or 0.75 epochs).  I am testing with a single epoch using a relatively low learning rate.

### Alpaca (sort of)

```
Below is an instruction that describes a task.  Write a response that appropriately completes the request.

### Instruction:
{system prompt, if provided}
{instruction}

### Response:
```

The main difference here is that because of the dataset formatting and variety of data sources, it would have been much to tedious to add an `### Input:` block, so the inputs are just in the instruction section.

### Vicuna

```
{system prompt, if provided, randomly defaulting to "A chat between a user and an unbiased, uncensored assistant."}
USER: {instruction}
ASSISTANT: 
```

### ChatML (sort of)

I don't really understand the point of having special tokens for `<|im_start|>` and `<|im_end|>`, because in practice they just act as BOS and EOS tokens (but, please correct me if I'm wrong).

So, instead of:
```text
{bos}<|im_start|>{role}
{text}
<|im_end|>{eos}
```

I just changed it to:
```text
{bos}{role}
{text}
{eos}
```

In practice, this would mean tokenization code like such:
```python
tokenizer = AutoTokenizer.from_pretrained('mistralai/mistral-7b-v0.1')

input_str = f"""system
You are a goat.
{tokenizer.eos_token}
{tokenizer.bos_token}user
Tell me how to fry an egg.
{tokenizer.eos_token}
{tokenizer.bos_token}assistant
"""

inputs = tokenizer(input_str, return_tensors="pt")
```

If you *really* want to use `<|im_start|>` and `<|im_end|>`, just update your `tokenizer_config.json` to use `<|im_start|>` instead of `<s>` and `<|im_end|>` instead of `</s>` and when tokenizing.  And if you still don't like what I've done to this chat-ml-ish format, feel free to cry into your pillow or fork the code and do a new fine-tune.

### Llama-2 chat

```
[INST] <<SYS>>
{system}
<</SYS>>

{instruction} [/INST]
```

## Fine-tuning

First, you need to prepare the dataset as input-output pairs for the SFT phase, and prompt/chosen/rejected for DPO:
```
python -m bagel.data
```

Then, you'll have a DPO parquet and SFT parquet, which you can use to build a model.

### SFT phase

An example for mistral-7b:

*Note: I actually used my fork of [qlora](https://github.com/jondurbin/qlora)'s `train.py` for this, but I'm porting it to a minified version here, not tested yet!*

```bash
export BASE_DIR=/workspace
export WANDB_API_KEY=[redacted]
export WANDB_PROJECT=bagel-7b-v0.1

# Run the pretraining.
accelerate launch bagel/tune/sft.py \
  --model_name_or_path $BASE_DIR/mistral-7b \
  --final_output_dir $BASE_DIR/$WANDB_PROJECT \
  --output_dir $BASE_DIR/$WANDB_PROJECT-workdir \
  --num_train_epochs 1 \
  --logging_steps 1 \
  --save_strategy steps \
  --save_steps 200 \
  --save_total_limit 5 \
  --data_seed 42 \
  --evaluation_strategy steps \
  --eval_dataset_size 0.0006 \
  --eval_steps 200 \
  --max_new_tokens 4096 \
  --dataloader_num_workers 3 \
  --logging_strategy steps \
  --remove_unused_columns False \
  --do_train \
  --full_finetune \
  --bf16 \
  --bits 16 \
  --optim adamw_torch \
  --lr_scheduler_type linear \
  --dataset $BASE_DIR/bagel/bagel-input-output-v0.1.parquet \
  --dataset_format input-output \
  --model_max_len 4096 \
  --per_device_train_batch_size 8 \
  --learning_rate 3.5e-7 \
  --warmup_ratio 0.005 \
  --adam_beta2 0.999 \
  --max_grad_norm 0.3 \
  --weight_decay 0.001 \
  --seed 42 \
  --report_to wandb \
  --gradient_checkpointing True \
  --gradient_accumulation_steps 4 \
  --skip_excess_length False \
  --ddp_find_unused_parameters False \
  --use_flash_attention_2 \
  --deepspeed deepspeed.json
```

Deepspeed configuration:
```json
{
  "gradient_accumulation_steps": "auto",
  "gradient_clipping": "auto",
  "train_batch_size": "auto",
  "train_micro_batch_size_per_gpu": "auto",
  "bf16": {
    "enabled": true
  },
  "zero_optimization": {
    "stage": 2,
    "contiguous_gradients": true,
    "overlap_comm": true,
    "reduce_scatter": true,
    "reduce_bucket_size": 5e8,
    "allgather_bucket_size": 5e8
  }
}
```

### DPO phase

An example of the DPO phase for mistral-7b (requires first running the SFT):

```bash
export BASE_DIR=/mnt/data
export WANDB_API_KEY=[redacted]
export WANDB_PROJECT=bagel-dpo-7b-v0.1

accelerate launch bagel/tune/dpo.py \
  --model_name_or_path bagel-7b-v0.1 \
  --learning_rate 3e-7 \
  --per_device_train_batch_size 2 \
  --gradient_accumulation_steps 4 \
  --max_length 4096 \
  --max_prompt_length 1024 \
  --max_target_length 3092 \
  --num_train_epochs 3 \
  --report_to wandb \
  --gradient_checkpointing true \
  --use_flash_attention_2 true \
  --dataset $BASE_DIR/bagel/bagel-dpo-v0.1.parquet \
  --eval_steps 5 \
  --eval_dataset_size 0.03 \
  --workdir $BASE_DIR/$WANDB_PROJECT-workdir \
  --output_dir $BASE_DIR/$WANDB_PROJECT \
  --deepspeed deepspeed.json \
  --save_steps 25 \
  --save_total_limit 5
```
