# yet-another-retnet

(Work in progress) A simple but robust PyTorch implementation of RetNet from [Retentive Network: A Successor to Transformer for Large Language Models](https://arxiv.org/pdf/2307.08621.pdf).

<img src="doc/retnet-scaling.jpeg" alt="compare-attention-mechanisms" width="600"/>


### TODO

- [x] Equivalent **parallel** and **recursive** retention methods.  See: [retention.py](yet_another_retnet/retention.py)
- [x] Recurrent position embedding implementation.
- [x] `MultiScaleRetention` module.  See: [retention.py](yet_another_retnet/retention.py)
- [x] Make relative position embeddings for `MultiScaleRetention` **optional**.
    - The retention layer explicitly includes a position embedding update, which is based on [xPos](https://arxiv.org/pdf/2212.10554.pdf).  It does not necessarily translate well to other domains (e.g. computer vision, heterogeneous graphs).  So I have made it optional.
    - I'm not 100% sure why the authors did this.  It seems overly specific to the language modeling use case, and it's not clear to me that it was necessary.
- [x] End-to-end `RetNet` module.
    - [x] `RetNetDecoderLayer`
    - [x] `RetNetDecoder`
- [x] Preconfigured 1.3B, 2.7B, and 6.7B models (untrained)
- [ ] Reproduce inference memory, throughput, and latency benchmarks.
- [ ] Equivalent **chunkwise** retention method.
- [ ] Release stable version on PyPI.
- [ ] Basic training example for language modeling.


## Install

```bash
pip install "yet-another-retnet @ git+ssh://git@github.com/fkodom/yet-another-retnet.git"
```

For contributors:
```bash
# Install all dev dependencies (tests etc.)
pip install "yet-another-retnet[test] @ git+ssh://git@github.com/fkodom/yet-another-retnet.git"
# Setup pre-commit hooks
pre-commit install
```


## About

RetNet is a transformer-like architecture that has equivalent **parallel** and **recurrent** formulations.

<img src="doc/retention-dual-forms.jpeg" alt="retention-dual-forms" width="600"/>


The benefits of this dual formulation are:
- Accuracy comparable to Transformer-based models
- **Parallel**: high training throughput
- **Recurrent**: high inference throughput

This is the "impossible triangle" of language model design, as described by the authors:

<img src="doc/impossible-triangle.jpeg" alt="impossible-triangle" width="350"/>


## Usage

### RetNet

Use one of the configurations described in the paper:
- `retnet_1_3b`
- `retnet_2_7b`
- `retnet_6_7b`

```python
from yet_another_retnet.retnet import retnet_1_3b

retnet = retnet_1_3b(num_tokens=10000, device="cuda")
```

or create your own `RetNet` model directly:

```python
from yet_another_retnet.retnet import RetNet

# a very small RetNet model :D
retnet = RetNet(
    num_tokens=1000, # vocab size, usually taken from tokenizer
    d_model=64,
    nhead=4,
    num_layers=2,
    device="cuda",
).eval()  # Important for reproducibility!
```

Equivalent parallel and recurrent usage:

```python
import torch

# Set deterministic CUDA ops
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False

# input shape: (batch_size, seq_len)
# integer range: [0, num_tokens)
x = torch.randint(0, 1000, (1, 16), device="cuda")

# Parallel usage
y_parallel = retnet.forward_parallel(x)

# Recurrent usage
outputs = []  # container for collecting step-wise outputs
prev_states = []  # cache layer states after each step
for idx in range(16):  # seq_len
    out, prev_states = retnet.forward_recurrent(x[:, idx], idx, prev_states)
    outputs.append(out)
y_recursive = torch.stack(outputs, dim=1)

# Check that outputs are equal
torch.testing.assert_close(y_parallel, y_recursive)
```

**NOTE**: There is some floating point error accumulation in the recurrent formulation, which I believe is less pronounced in the parallel formulation. Especially for untrained models (when activations are very large), the two outputs may not match *exactly*.  The difference should still be very small -- on the order of 1e-5 or less.


### MultiScaleRetention

Equivalent parallel and recurrent usage:

```python
import torch

from yet_another_retnet.retention import MultiScaleRetention

mhr = MultiScaleRetention(embed_dim=32, num_heads=4, device="cuda").eval()

# input shape: (batch_size, seq_len, embed_dim)
q = k = v = torch.randn(1, 16, 32, device="cuda")

# Parallel retention
y_parallel, _ = mhr.forward_parallel(q, k, v)

# Recursive retention
outputs = []
prev_state = None
for idx in range(32):
    out, prev_state = mhr.forward_recurrent(
        q[:, idx], k[:, idx], v[:, idx], idx, prev_state
    )
    outputs.append(out)
y_recursive = torch.stack(outputs, dim=1)

# Check that outputs are equal
torch.testing.assert_close(y_parallel, y_recursive)
```

**NOTE**: The `MultiScaleRetention` that is described in the paper includes an
explicit position embedding (based on xPos) as part of the retention layer.  This
does not translate perfectly to other domains (e.g. computer vision, heterogeneous
graphs), so I have made it optional.

Set `relative_position=False` to disable the position embedding.  Instead, you will
be responsible for adding positional information to the inputs (if needed).

```python
# Disable relative position embedding
mhr = MultiScaleRetention(
    embed_dim=32, num_heads=4, relative_position=False, device="cuda"
)
# Everything else works the same as above.
# Just add your own positional embeddings to the inputs.
```

### Retention forward pass

Similar to the example above, but head projections and positional updates are not internalized by `MultiScaleRetention`:

```python
import torch

from yet_another_retnet.retention import retention_parallel, retention_recurrent

# input shape: (batch_size, num_heads, seq_len, head_dim)
q = k = v = torch.randn(1, 4, 32, 8, device="cuda")

# Parallel retention
y_parallel, _ = retention_parallel(q, k, v)

# Recursive retention
outputs = []
prev_state = None
for i in range(32):
    out, prev_state = retention_recurrent(q[:, :, i], k[:, :, i], v[:, :, i], prev_state)
    outputs.append(out)
y_recursive = torch.stack(outputs, dim=2)
```

