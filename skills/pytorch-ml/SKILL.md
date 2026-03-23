---
name: pytorch-ml
description: PyTorch model loading, training loop with mixed precision, DataLoader, checkpointing, and inference optimization
license: MIT
triggers:
  - pytorch training
  - pytorch model
  - train neural network pytorch
  - pytorch dataloader
  - mixed precision training
  - pytorch checkpoint
  - huggingface model pytorch
  - pytorch inference
  - pytorch gradient clipping
  - torch cuda amp
metadata:
  skill-author: gonzih
  data-sources:
    - huggingface.co/models — pretrained model weights; free; optional HF_TOKEN for gated models
    - local checkpoint files — .pt/.pth files
  byok:
    - HF_TOKEN — optional; required for gated HuggingFace models (Llama, Gemma etc.)
  compatibility: claude-code>=1.0
---

# PyTorch ML Skill

## What it does
Covers the complete PyTorch ML workflow: loading models from files or HuggingFace Hub, building custom Dataset/DataLoader pipelines, writing a production-grade training loop with gradient clipping and mixed precision (torch.cuda.amp), saving and resuming from checkpoints, and optimizing inference with torch.no_grad and model.eval().

## How to invoke
- "Write a PyTorch training loop with mixed precision"
- "Load a pretrained ResNet model and fine-tune it"
- "Create a custom PyTorch Dataset class for tabular data"
- "Add checkpoint save/resume to my training loop"
- "Set up mixed precision training with torch.cuda.amp"
- "Optimize my PyTorch model for inference"

## Key parameters
- `model` — nn.Module subclass
- `device` — `cuda`, `cpu`, or `mps` (Apple Silicon)
- `batch_size` — DataLoader batch size
- `num_workers` — parallel data loading workers
- `amp_enabled` — mixed precision (`torch.float16` on CUDA)
- `grad_clip_norm` — max gradient norm for clipping (default 1.0)
- `checkpoint_path` — path to save/load .pt checkpoint file

## Rate limits
- HuggingFace Hub: no hard limit, but large model downloads may be throttled
- Local training: depends only on hardware

## Workflow steps
1. Install: `pip install torch torchvision`
2. Define Dataset and wrap in DataLoader
3. Initialize model, optimizer, scheduler
4. Set up GradScaler for mixed precision
5. Training loop: zero_grad → forward → loss → scaler.scale → backward → unscale → clip → scaler.step → update
6. Save checkpoint after each epoch
7. For inference: model.eval() + torch.no_grad()

## Working code example

```python
import os
import json
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader, random_split
from torch.cuda.amp import GradScaler, autocast
from pathlib import Path
from typing import Optional, Callable
import numpy as np


# ── Device setup ─────────────────────────────────────────────────────────────

def get_device() -> torch.device:
    """Get the best available device."""
    if torch.cuda.is_available():
        return torch.device("cuda")
    elif torch.backends.mps.is_available():
        return torch.device("mps")  # Apple Silicon
    return torch.device("cpu")


# ── Custom Dataset ─────────────────────────────────────────────────────────────

class TabularDataset(Dataset):
    """
    Custom Dataset for tabular data from numpy arrays.
    Works with any 2D feature array + 1D label array.
    """
    def __init__(self, features: np.ndarray, labels: np.ndarray,
                 transform: Optional[Callable] = None):
        self.features = torch.tensor(features, dtype=torch.float32)
        self.labels = torch.tensor(labels, dtype=torch.long)
        self.transform = transform

    def __len__(self) -> int:
        return len(self.features)

    def __getitem__(self, idx: int):
        x = self.features[idx]
        y = self.labels[idx]
        if self.transform:
            x = self.transform(x)
        return x, y


def create_dataloaders(
    dataset: Dataset,
    batch_size: int = 32,
    val_split: float = 0.2,
    num_workers: int = 4,
    pin_memory: bool = True,
) -> tuple[DataLoader, DataLoader]:
    """Split dataset into train/val and create DataLoaders."""
    n_val = int(len(dataset) * val_split)
    n_train = len(dataset) - n_val
    train_ds, val_ds = random_split(dataset, [n_train, n_val],
                                    generator=torch.Generator().manual_seed(42))
    train_loader = DataLoader(
        train_ds,
        batch_size=batch_size,
        shuffle=True,
        num_workers=num_workers,
        pin_memory=pin_memory and torch.cuda.is_available(),
        drop_last=True,
        persistent_workers=num_workers > 0,
    )
    val_loader = DataLoader(
        val_ds,
        batch_size=batch_size * 2,
        shuffle=False,
        num_workers=num_workers,
        pin_memory=pin_memory and torch.cuda.is_available(),
        persistent_workers=num_workers > 0,
    )
    return train_loader, val_loader


# ── Example model ─────────────────────────────────────────────────────────────

class MLP(nn.Module):
    """Simple multi-layer perceptron with BatchNorm and Dropout."""
    def __init__(self, input_dim: int, hidden_dims: list[int], num_classes: int,
                 dropout: float = 0.3):
        super().__init__()
        layers = []
        in_dim = input_dim
        for hidden_dim in hidden_dims:
            layers.extend([
                nn.Linear(in_dim, hidden_dim),
                nn.BatchNorm1d(hidden_dim),
                nn.ReLU(inplace=True),
                nn.Dropout(dropout),
            ])
            in_dim = hidden_dim
        layers.append(nn.Linear(in_dim, num_classes))
        self.net = nn.Sequential(*layers)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.net(x)


# ── Checkpoint utilities ──────────────────────────────────────────────────────

def save_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: optim.Optimizer,
    scheduler,
    scaler: GradScaler,
    epoch: int,
    best_val_loss: float,
    metrics: dict,
) -> None:
    """Save full training state for resume."""
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "scheduler_state_dict": scheduler.state_dict() if scheduler else None,
        "scaler_state_dict": scaler.state_dict(),
        "best_val_loss": best_val_loss,
        "metrics": metrics,
    }, path)


def load_checkpoint(
    path: str,
    model: nn.Module,
    optimizer: Optional[optim.Optimizer] = None,
    scheduler=None,
    scaler: Optional[GradScaler] = None,
    device: Optional[torch.device] = None,
) -> dict:
    """Load checkpoint, restore model and optionally optimizer/scheduler state."""
    device = device or get_device()
    checkpoint = torch.load(path, map_location=device, weights_only=True)
    model.load_state_dict(checkpoint["model_state_dict"])
    if optimizer and "optimizer_state_dict" in checkpoint:
        optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
    if scheduler and checkpoint.get("scheduler_state_dict"):
        scheduler.load_state_dict(checkpoint["scheduler_state_dict"])
    if scaler and "scaler_state_dict" in checkpoint:
        scaler.load_state_dict(checkpoint["scaler_state_dict"])
    return checkpoint


# ── Training loop ─────────────────────────────────────────────────────────────

def train_one_epoch(
    model: nn.Module,
    loader: DataLoader,
    optimizer: optim.Optimizer,
    criterion: nn.Module,
    scaler: GradScaler,
    device: torch.device,
    grad_clip_norm: float = 1.0,
    amp_enabled: bool = True,
) -> dict:
    """
    One training epoch with mixed precision and gradient clipping.
    Returns dict of training metrics.
    """
    model.train()
    total_loss = 0.0
    correct = 0
    total = 0

    for batch_idx, (inputs, targets) in enumerate(loader):
        inputs = inputs.to(device, non_blocking=True)
        targets = targets.to(device, non_blocking=True)

        optimizer.zero_grad(set_to_none=True)  # faster than zero_grad()

        # Mixed precision forward pass
        with autocast(device_type=device.type, enabled=amp_enabled and device.type == "cuda"):
            outputs = model(inputs)
            loss = criterion(outputs, targets)

        # Scaled backward pass
        scaler.scale(loss).backward()

        # Unscale before gradient clipping
        scaler.unscale_(optimizer)
        nn.utils.clip_grad_norm_(model.parameters(), max_norm=grad_clip_norm)

        # Optimizer step via scaler
        scaler.step(optimizer)
        scaler.update()

        total_loss += loss.item() * inputs.size(0)
        preds = outputs.argmax(dim=1)
        correct += preds.eq(targets).sum().item()
        total += inputs.size(0)

    return {
        "loss": total_loss / total,
        "accuracy": correct / total,
    }


@torch.no_grad()
def evaluate(
    model: nn.Module,
    loader: DataLoader,
    criterion: nn.Module,
    device: torch.device,
) -> dict:
    """Evaluate model on a DataLoader. No gradient computation."""
    model.eval()
    total_loss = 0.0
    correct = 0
    total = 0

    for inputs, targets in loader:
        inputs = inputs.to(device, non_blocking=True)
        targets = targets.to(device, non_blocking=True)
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        total_loss += loss.item() * inputs.size(0)
        correct += outputs.argmax(1).eq(targets).sum().item()
        total += inputs.size(0)

    return {"loss": total_loss / total, "accuracy": correct / total}


def train(
    model: nn.Module,
    train_loader: DataLoader,
    val_loader: DataLoader,
    num_epochs: int = 50,
    lr: float = 1e-3,
    weight_decay: float = 1e-4,
    checkpoint_dir: str = "checkpoints",
    resume_from: Optional[str] = None,
    amp_enabled: bool = True,
    grad_clip_norm: float = 1.0,
) -> dict:
    """
    Full training loop with:
    - Mixed precision (AMP)
    - Gradient clipping
    - LR scheduling (CosineAnnealingLR)
    - Checkpoint save/resume
    - Best model tracking
    """
    device = get_device()
    model = model.to(device)
    criterion = nn.CrossEntropyLoss(label_smoothing=0.1)
    optimizer = optim.AdamW(model.parameters(), lr=lr, weight_decay=weight_decay)
    scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=num_epochs, eta_min=lr/100)
    scaler = GradScaler(enabled=amp_enabled and device.type == "cuda")

    Path(checkpoint_dir).mkdir(parents=True, exist_ok=True)
    start_epoch = 0
    best_val_loss = float("inf")
    history = {"train_loss": [], "val_loss": [], "train_acc": [], "val_acc": []}

    # Resume from checkpoint
    if resume_from and Path(resume_from).exists():
        ckpt = load_checkpoint(resume_from, model, optimizer, scheduler, scaler, device)
        start_epoch = ckpt["epoch"] + 1
        best_val_loss = ckpt.get("best_val_loss", float("inf"))
        history = ckpt.get("metrics", history)
        print(f"Resumed from epoch {ckpt['epoch']}, best_val_loss={best_val_loss:.4f}")

    print(f"Training on {device} | AMP: {amp_enabled and device.type == 'cuda'}")

    for epoch in range(start_epoch, num_epochs):
        train_metrics = train_one_epoch(
            model, train_loader, optimizer, criterion, scaler, device,
            grad_clip_norm=grad_clip_norm, amp_enabled=amp_enabled,
        )
        val_metrics = evaluate(model, val_loader, criterion, device)
        scheduler.step()

        history["train_loss"].append(train_metrics["loss"])
        history["val_loss"].append(val_metrics["loss"])
        history["train_acc"].append(train_metrics["accuracy"])
        history["val_acc"].append(val_metrics["accuracy"])

        lr_now = scheduler.get_last_lr()[0]
        print(f"Epoch {epoch+1:3d}/{num_epochs} | "
              f"train_loss={train_metrics['loss']:.4f} acc={train_metrics['accuracy']:.3f} | "
              f"val_loss={val_metrics['loss']:.4f} acc={val_metrics['accuracy']:.3f} | "
              f"lr={lr_now:.2e}")

        # Save latest checkpoint
        save_checkpoint(
            f"{checkpoint_dir}/latest.pt", model, optimizer, scheduler, scaler,
            epoch, best_val_loss, history,
        )

        # Save best checkpoint
        if val_metrics["loss"] < best_val_loss:
            best_val_loss = val_metrics["loss"]
            save_checkpoint(
                f"{checkpoint_dir}/best.pt", model, optimizer, scheduler, scaler,
                epoch, best_val_loss, history,
            )
            print(f"  ✓ New best model saved (val_loss={best_val_loss:.4f})")

    return history


# ── Loading from HuggingFace Hub ──────────────────────────────────────────────

def load_hf_model(model_name: str, task: str = "feature-extraction"):
    """
    Load a model from HuggingFace Hub.
    Set HF_TOKEN env var for gated models (Llama, Gemma, etc.).
    """
    try:
        from transformers import AutoModel, AutoTokenizer, pipeline
    except ImportError:
        raise ImportError("pip install transformers")

    token = os.environ.get("HF_TOKEN")
    tokenizer = AutoTokenizer.from_pretrained(model_name, token=token)
    model = AutoModel.from_pretrained(model_name, token=token, torch_dtype=torch.float16)
    device = get_device()
    model = model.to(device)
    model.eval()
    return model, tokenizer


# ── Inference optimization ─────────────────────────────────────────────────────

@torch.no_grad()
def batch_inference(
    model: nn.Module,
    data: torch.Tensor,
    device: torch.device,
    batch_size: int = 256,
) -> torch.Tensor:
    """
    Run inference on a large tensor in batches.
    model.eval() + torch.no_grad() = no gradients tracked.
    """
    model.eval()
    model = model.to(device)
    results = []
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size].to(device)
        with autocast(device_type=device.type, enabled=device.type == "cuda"):
            output = model(batch)
        results.append(output.cpu())
    return torch.cat(results, dim=0)


# --- Example usage ---
if __name__ == "__main__":
    device = get_device()
    print(f"Using device: {device}")

    # Create synthetic dataset
    np.random.seed(42)
    X = np.random.randn(1000, 20).astype(np.float32)
    y = (X[:, 0] + X[:, 1] > 0).astype(np.int64)

    dataset = TabularDataset(X, y)
    train_loader, val_loader = create_dataloaders(dataset, batch_size=32, num_workers=0)
    print(f"Train batches: {len(train_loader)}, Val batches: {len(val_loader)}")

    model = MLP(input_dim=20, hidden_dims=[64, 32], num_classes=2, dropout=0.2)
    print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")

    # Train for 5 epochs as demo
    history = train(
        model, train_loader, val_loader,
        num_epochs=5,
        lr=1e-3,
        checkpoint_dir="/tmp/demo_checkpoints",
        amp_enabled=False,  # AMP only beneficial on CUDA
    )

    print(f"\nFinal val accuracy: {history['val_acc'][-1]:.3f}")
    print(f"Best val loss: {min(history['val_loss']):.4f}")
```

## Live Data Sources
- **PyTorch documentation**: https://pytorch.org/docs/stable/
- **HuggingFace Hub**: https://huggingface.co/models — pretrained models
- **TorchVision models**: https://pytorch.org/vision/stable/models.html
- **PyTorch tutorials**: https://pytorch.org/tutorials/
- **Mixed precision guide**: https://pytorch.org/docs/stable/amp.html
- **DataLoader guide**: https://pytorch.org/docs/stable/data.html
