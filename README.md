# Le Chat Atelier

Generate SVG illustrations from descriptions with Ministral 3 3B and a text-only LoRA adapter. The experiment uses the [BF16 instruct model](https://huggingface.co/mistralai/Ministral-3-3B-Instruct-2512-BF16) and [OmniSVG/MMSVG-Illustration](https://huggingface.co/datasets/OmniSVG/MMSVG-Illustration), licensed CC BY-NC-SA 4.0. A [transcript of the development session](CHAT.md) accompanies the source.

## Data

Install PyArrow, CairoSVG (with native Cairo), Pillow and tokenizers. Download the first two training Parquet shards and the model's tokenizer.json. Run:

```sh
python prep.py train-00000-of-00026.parquet train-00001-of-00026.parquet --tokenizer tokenizer.json --output data
```

Seed 42, 7,317 candidates examined, 5,000 training examples (22,012,500 tokens), 50 validation examples and 20 fixed evaluation prompts. SVGs have multiple fill colours, a 200×200 viewBox and paths rounded to one decimal place. Every accepted target rendered nonblank on white. Exact duplicate cleaned targets are removed before splitting.

## Training and serving

The instance needs CUDA PyTorch, Transformers with Ministral 3 support, PEFT, Accelerate, huggingface_hub and tokenizers. Run each process from the project directory on its assigned GPU.

```sh
CUDA_VISIBLE_DEVICES=0 python -u train.py --data data --minutes 180
CUDA_VISIBLE_DEVICES=1 python -u demo.py
```

Training uses rank 32, alpha 64, learning rate 0.0001, 8 examples per optimizer update, one example per forward pass, completion-only token-weighted loss and a single shuffled pass capped by elapsed training time. A checkpoint becomes selectable once its adapter finishes saving. Each checkpoint can be loaded for further adaptation with PEFT's `is_trainable=True`; it contains adapter weights, without optimizer state.

The web server listens on the instance's 127.0.0.1:8090. Use SSH local port forwarding to reach it from your laptop. The model picker switches between base and the latest available adapter. The preview updates as complete SVG elements arrive.

## Evaluation

From a machine with CairoSVG, Pillow and access to the forwarded port:

```sh
python eval.py --prompts data/eval.jsonl --model base --out results/base
python eval.py --prompts data/eval.jsonl --model step-0050 --out results/step-0050
```

The two runs use identical prompts and greedy decoding. Each output records model time, generated tokens, SVG closure, nonblank rendering and latency to the first complete path when available. Inspect each contact sheet beside the descriptions for semantic match. Raw model responses are saved beside the PNGs.

## Mac studio

If port 8090 is already forwarded, double-click `Open Atelier.command`. Otherwise run the launcher from Terminal with your SSH settings, for example `ATELIER_SSH_HOST=user@example.com ATELIER_SSH_PORT=22 "./Open Atelier.command"`. It opens the local studio in Helium. The interface runs at `http://127.0.0.1:8765` and uses an SSH tunnel to the existing Vast rental. The default model is the final step-625 adapter. Press Command–Enter to send a description. Recent completed drawings are stored in this browser on your Mac.
