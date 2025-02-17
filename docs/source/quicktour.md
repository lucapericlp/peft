<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.

⚠️ Note that this file is in Markdown but contain specific syntax for our doc-builder (similar to MDX) that may not be
rendered properly in your Markdown viewer.

-->

# Quicktour

🤗 PEFT contains parameter-efficient finetuning methods for training large pretrained models. The traditional paradigm is to finetune all of a model's parameters for each downstream task, but this is becoming exceedingly costly and impractical because of the enormous number of parameters in models today. Instead, it is more efficient to train a smaller number of prompt parameters or use a reparametrization method like low-rank adaptation (LoRA) to reduce the number of trainable parameters. 

This quicktour will show you 🤗 PEFT's main features and help you train large pretrained models that would typically be inaccessible on consumer devices. You'll see how to train the 1.2B parameter [`bigscience/mt0-large`](https://huggingface.co/bigscience/mt0-large) model with LoRA to generate a classification label and use it for inference.

## PeftConfig

Each 🤗 PEFT method is defined by a [`PeftConfig`] class that stores all the important parameters for building a [`PeftModel`]. 

Because you're going to use LoRA, you'll need to load and create a [`LoraConfig`] class. Within `LoraConfig`, specify the following parameters:

- the `task_type`, or sequence-to-sequence language modeling in this case
- `inference_mode`, whether you're using the model for inference or not
- `r`, the dimension of the low-rank matrices
- `lora_alpha`, the scaling factor for the low-rank matrices
- `lora_dropout`, the dropout probability of the LoRA layers

```python
from peft import LoraConfig, TaskType

peft_config = LoraConfig(task_type=TaskType.SEQ_2_SEQ_LM, inference_mode=False, r=8, lora_alpha=32, lora_dropout=0.1)
```

<Tip>

💡 See the [`LoraConfig`] reference for more details about other parameters you can adjust.

</Tip>

## PeftModel

A [`PeftModel`] is created by the [`get_peft_model`] function. It takes a base model - which you can load from the 🤗 Transformers library - and the [`PeftConfig`] containing the instructions for how to configure a model for a specific 🤗 PEFT method.

Start by loading the base model you want to finetune.

```python
from transformers import AutoModelForSeq2SeqLM

model_name_or_path = "bigscience/mt0-large"
tokenizer_name_or_path = "bigscience/mt0-large"
model = AutoModelForSeq2SeqLM.from_pretrained(model_name_or_path)
```

Wrap your base model and `peft_config` with the `get_peft_model` function to create a [`PeftModel`]. To get a sense of the number of trainable parameters in your model, use the [`print_trainable_parameters`] method. In this case, you're only training 0.19% of the model's parameters! 🤏

```python
from peft import get_peft_model

model = get_peft_model(model, peft_config)
model.print_trainable_parameters()
"output: trainable params: 2359296 || all params: 1231940608 || trainable%: 0.19151053100118282"
```

That is it 🎉! Now you can train the model using the 🤗 Transformers [`~transformers.Trainer`], 🤗 Accelerate, or any custom PyTorch training loop.

## Save and load a model

After your model is finished training, you can save your model to a directory using the [`~transformers.PreTrainedModel.save_pretrained`] function. You can also save your model to the Hub (make sure you log in to your Hugging Face account first) with the [`~transformers.PreTrainedModel.push_to_hub`] function.

```python
model.save_pretrained("output_dir")

# if pushing to Hub
from huggingface_hub import notebook_login

notebook_login()
model.push_to_hub("my_awesome_peft_model")
```

This only saves the incremental 🤗 PEFT weights that were trained, meaning it is super efficient to store, transfer, and load. For example, this [`bigscience/T0_3B`](https://huggingface.co/smangrul/twitter_complaints_bigscience_T0_3B_LORA_SEQ_2_SEQ_LM) model trained with LoRA on the [`twitter_complaints`](https://huggingface.co/datasets/ought/raft/viewer/twitter_complaints/train) subset of the RAFT [dataset](https://huggingface.co/datasets/ought/raft) only contains two files: `adapter_config.json` and `adapter_model.bin`. The latter file is just 19MB!

Easily load your model for inference using the [`~transformers.PreTrainedModel.from_pretrained`] function:

```diff
  from transformers import AutoModelForSeq2SeqLM
+ from peft import PeftModel, PeftConfig

+ peft_model_id = "smangrul/twitter_complaints_bigscience_T0_3B_LORA_SEQ_2_SEQ_LM"
+ config = PeftConfig.from_pretrained(peft_model_id)
  model = AutoModelForSeq2SeqLM.from_pretrained(config.base_model_name_or_path)
+ model = PeftModel.from_pretrained(model, peft_model_id)
  tokenizer = AutoTokenizer.from_pretrained(config.base_model_name_or_path)

  model = model.to(device)
  model.eval()
  inputs = tokenizer("Tweet text : @HondaCustSvc Your customer service has been horrible during the recall process. I will never purchase a Honda again. Label :", return_tensors="pt")

  with torch.no_grad():
      outputs = model.generate(input_ids=inputs["input_ids"].to("cuda"), max_new_tokens=10)
      print(tokenizer.batch_decode(outputs.detach().cpu().numpy(), skip_special_tokens=True)[0])
  'complaint'
```

## Easy loading with Auto classes 

If you have saved your adapter locally or on the Hub, you can leverage the `AutoPeftModelForxxx` classes and load any PEFT model with a single line of code:

```diff
- from peft import PeftConfig, PeftModel
- from transformers import AutoModelForCausalLM
+ from peft import AutoPeftModelForCausalLM

- peft_config = PeftConfig.from_pretrained("ybelkada/opt-350m-lora") 
- base_model_path = peft_config.base_model_name_or_path
- transformers_model = AutoModelForCausalLM.from_pretrained(base_model_path)
- peft_model = PeftModel.from_pretrained(transformers_model, peft_config)
+ peft_model = AutoPeftModelForCausalLM.from_pretrained("ybelkada/opt-350m-lora")
```

Currently, supported auto classes are: `AutoPeftModelForCausalLM`, `AutoPeftModelForSequenceClassification`, `AutoPeftModelForSeq2SeqLM`, `AutoPeftModelForTokenClassification`, `AutoPeftModelForQuestionAnswering` and `AutoPeftModelForFeatureExtraction`. For other tasks (e.g. Whisper, StableDiffusion), you can load the model with:

```diff
- from peft import PeftModel, PeftConfig, AutoPeftModel
+ from peft import AutoPeftModel
- from transformers import WhisperForConditionalGeneration

- model_id = "smangrul/openai-whisper-large-v2-LORA-colab"

peft_model_id = "smangrul/openai-whisper-large-v2-LORA-colab"
- peft_config = PeftConfig.from_pretrained(peft_model_id)
- model = WhisperForConditionalGeneration.from_pretrained(
-     peft_config.base_model_name_or_path, load_in_8bit=True, device_map="auto"
- )
- model = PeftModel.from_pretrained(model, peft_model_id)
+ model = AutoPeftModel.from_pretrained(peft_model_id)
```

## Next steps

Now that you've seen how to train a model with one of the 🤗 PEFT methods, we encourage you to try out some of the other methods like prompt tuning. The steps are very similar to the ones shown in this quickstart; prepare a [`PeftConfig`] for a 🤗 PEFT method, and use the `get_peft_model` to create a [`PeftModel`] from the configuration and base model. Then you can train it however you like!

Feel free to also take a look at the task guides if you're interested in training a model with a 🤗 PEFT method for a specific task such as semantic segmentation, multilingual automatic speech recognition, DreamBooth, and token classification.