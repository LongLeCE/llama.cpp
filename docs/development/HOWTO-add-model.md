# Add a new model architecture to `llama.cpp`

Adding a model requires few steps:

1. Convert the model to GGUF
2. Define the model architecture in `llama.cpp`
3. Build the GGML graph implementation

After following these steps, you can open PR.

Also, it is important to check that the examples and main ggml backends (CUDA, METAL, CPU) are working with the new architecture, especially:
- [main](/tools/main/)
- [imatrix](/tools/imatrix/)
- [quantize](/tools/quantize/)
- [server](/tools/server/)

### 1. Convert the model to GGUF

This step is done in python with a `convert` script using the [gguf](https://pypi.org/project/gguf/) library.
Depending on the model architecture, you can use either [convert_hf_to_gguf.py](/convert_hf_to_gguf.py) or [examples/convert_legacy_llama.py](/examples/convert_legacy_llama.py) (for `llama/llama2` models in `.pth` format).

The convert script reads the model configuration, tokenizer, tensor names+data and converts them to GGUF metadata and tensors.

The required steps to implement for an HF model are:

1. Define the model `ModelBase.register` annotation in a new `TextModel` or `MmprojModel` subclass, example:

```python
@ModelBase.register("MyModelForCausalLM")
class MyModel(TextModel):
    model_arch = gguf.MODEL_ARCH.MYMODEL
```

or

```python
@ModelBase.register("MyModelForConditionalGeneration")
class MyModel(MmprojModel):
    model_arch = gguf.MODEL_ARCH.MYMODEL
```

2. Define the layout of the GGUF tensors in [constants.py](/gguf-py/gguf/constants.py)

Add an enum entry in `MODEL_ARCH`, the model human friendly name in `MODEL_ARCH_NAMES` and the GGUF tensor names in `MODEL_TENSORS`.

Example for `falcon` model:
```python
    MODEL_ARCH.FALCON: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_NORM_2,
        MODEL_TENSOR.ATTN_QKV,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
    ]
```

3. Map the original tensor names to the standardize equivalent in GGUF

As a general rule, before adding a new tensor name to GGUF, be sure the equivalent naming does not already exist.

Once you have found the GGUF tensor name equivalent, add it to the [tensor_mapping.py](/gguf-py/gguf/tensor_mapping.py) file.

If the tensor name is part of a repetitive layer/block, the key word `bid` substitutes it.

Example for the normalization tensor in attention layers:

```python
block_mappings_cfg: dict[MODEL_TENSOR, tuple[str, ...]] = {
        # Attention norm
        MODEL_TENSOR.ATTN_NORM: (
            "gpt_neox.layers.{bid}.input_layernorm",                # gptneox
            "transformer.h.{bid}.ln_1",                             # gpt2 gpt-j refact qwen
            "transformer.blocks.{bid}.norm_1",                      # mpt
            ...
        )
}
```

`transformer.blocks.{bid}.norm_1` will be mapped to `blk.{bid}.attn_norm` in GGUF.

Depending on the model configuration, tokenizer, code and tensors layout, you will have to override:
- `TextModel#set_gguf_parameters`
- `MmprojModel#set_gguf_parameters`
- `ModelBase#set_vocab`
- `ModelBase#modify_tensors`

NOTE: Tensor names must end with `.weight` or `.bias` suffixes, that is the convention and several tools like `quantize` expect this to proceed the weights.

### 2. Define the model architecture in `llama.cpp`

The model params and tensors layout must be defined in `llama.cpp` source files:
1. Define a new `llm_arch` enum value in `src/llama-arch.h`.
2. In `src/llama-arch.cpp`:
    - Add the architecture name to the `LLM_ARCH_NAMES` map.
    - Add the tensor mappings to the `LLM_TENSOR_NAMES` map.
3. Add any non-standard metadata loading in the `llama_model_loader` constructor in `src/llama-model-loader.cpp`.
4. If the model has a RoPE operation, add a case for the architecture in `llama_model_rope_type` function in `src/llama-model.cpp`.

NOTE: The dimensions in `ggml` are typically in the reverse order of the `pytorch` dimensions.

### 3. Build the GGML graph implementation

This is the funniest part, you have to provide the inference graph implementation of the new model architecture in `src/llama-model.cpp`.
Create a new struct that inherits from `llm_graph_context` and implement the graph-building logic in its constructor.
Have a look at existing implementations like `llm_build_llama`, `llm_build_dbrx` or `llm_build_bert`.
Then, in the `llama_model::build_graph` method, add a case for your architecture to instantiate your new graph-building struct.

Some `ggml` backends do not support all operations. Backend implementations can be added in a separate PR.

Note: to debug the inference graph: you can use [llama-eval-callback](/examples/eval-callback/).

## GGUF specification

https://github.com/ggml-org/ggml/blob/master/docs/gguf.md

## Resources

- YaRN RoPE scaling https://github.com/ggml-org/llama.cpp/pull/2268
- support Baichuan serial models https://github.com/ggml-org/llama.cpp/pull/3009
- support attention bias https://github.com/ggml-org/llama.cpp/pull/4283
- Mixtral support https://github.com/ggml-org/llama.cpp/pull/4406
- BERT embeddings https://github.com/ggml-org/llama.cpp/pull/5423
- Grok-1 support https://github.com/ggml-org/llama.cpp/pull/6204
- Command R Plus support https://github.com/ggml-org/llama.cpp/pull/6491
- support arch DBRX https://github.com/ggml-org/llama.cpp/pull/6515
- How to convert HuggingFace model to GGUF format https://github.com/ggml-org/llama.cpp/discussions/2948
