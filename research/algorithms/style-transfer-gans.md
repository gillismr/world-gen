# Style Transfer & GANs for Terrain Generation

## Overview

Machine learning approaches — primarily **Generative Adversarial Networks (GANs)** and **neural style transfer** — are an emerging but increasingly practical tool in procedural terrain generation. Rather than hand-crafting simulation rules, these methods learn to generate or transform terrain by training on real data (satellite DEMs, geological maps, climate datasets).

This is the most research-frontier entry in our algorithm list. It's not yet mainstream in game-dev world-gen but is actively studied and increasingly accessible.

---

## Neural Style Transfer (Gatys et al., 2015)

### Core Idea

Transfer the **style** (texture, pattern) of a reference image onto the **content** of another image. Applied to terrain: take a heightmap with uninteresting noise and transfer the visual style of a realistic DEM (Digital Elevation Model) onto it.

### How It Works

Given:
- Content image C (your generated heightmap)
- Style image S (real satellite terrain or a target art style)

Optimize a new image I to minimize:

```
L_total(C, S, I) = α * L_content(C, I) + β * L_style(S, I)

L_content = sum over layers: ||F_l(I) - F_l(C)||²     (feature maps from CNN)
L_style   = sum over layers: ||G_l(I) - G_l(S)||²     (Gram matrices of feature maps)

Gram matrix: G_l[i,j] = (1/N²) * sum_k: F_l[i,k] * F_l[j,k]
```

The Gram matrix captures texture statistics (correlations between feature channels) without caring about spatial layout — this separates "style" from "content."

```python
import torch
import torchvision.models as models
import torch.nn.functional as F

def gram_matrix(features):
    """Compute Gram matrix for style loss."""
    b, c, h, w = features.size()
    F = features.view(b, c, h * w)
    G = torch.bmm(F, F.transpose(1, 2))
    return G / (c * h * w)

def style_transfer(content_img, style_img, n_steps=300, alpha=1.0, beta=1e6):
    """Minimal neural style transfer."""
    vgg = models.vgg19(pretrained=True).features.eval()
    style_layers   = ['0','5','10','19','28']   # VGG conv layers
    content_layers = ['21']

    # Extract targets
    with torch.no_grad():
        style_features   = extract_features(style_img, vgg, style_layers)
        content_features = extract_features(content_img, vgg, content_layers)

    style_grams = {l: gram_matrix(f) for l, f in style_features.items()}

    # Optimizable image (start from content)
    generated = content_img.clone().requires_grad_(True)
    optimizer = torch.optim.LBFGS([generated])

    for step in range(n_steps):
        def closure():
            optimizer.zero_grad()
            gen_features = extract_features(generated, vgg, style_layers + content_layers)

            # Content loss
            content_loss = sum(
                F.mse_loss(gen_features[l], content_features[l])
                for l in content_layers
            )

            # Style loss
            style_loss = sum(
                F.mse_loss(gram_matrix(gen_features[l]), style_grams[l])
                for l in style_layers
            )

            loss = alpha * content_loss + beta * style_loss
            loss.backward()
            return loss

        optimizer.step(closure)

    return generated.detach()
```

---

## GANs — Generative Adversarial Networks

### Core Concept (Goodfellow et al., 2014)

Two networks compete:
- **Generator G**: takes noise z → generates fake data
- **Discriminator D**: distinguishes real data from fake

Training is a minimax game:

```
min_G max_D  E[log D(x)] + E[log(1 - D(G(z)))]

Discriminator loss: -( log D(x) + log(1 - D(G(z))) )
Generator loss:     -log D(G(z))
```

After training, G can produce new data indistinguishable from the training set.

### Conditional GAN (cGAN) — Most Useful for World-Gen

A conditional GAN takes both noise and a **conditioning signal** (e.g. a sketch map, a biome mask, or a tectonic outline) and generates terrain matching the condition:

```python
import torch
import torch.nn as nn

class Generator(nn.Module):
    def __init__(self, z_dim=100, condition_channels=1, out_channels=1):
        super().__init__()
        # Condition is a coarse map (biome mask, elevation sketch, etc.)
        self.encoder = nn.Sequential(
            nn.Conv2d(condition_channels, 64, 4, 2, 1),
            nn.LeakyReLU(0.2),
            nn.Conv2d(64, 128, 4, 2, 1),
            nn.BatchNorm2d(128),
            nn.LeakyReLU(0.2),
        )
        self.decoder = nn.Sequential(
            nn.ConvTranspose2d(128 + z_dim, 64, 4, 2, 1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.ConvTranspose2d(64, out_channels, 4, 2, 1),
            nn.Tanh()  # output in [-1, 1]
        )

    def forward(self, z, condition):
        cond_features = self.encoder(condition)
        z_expanded = z.view(z.size(0), -1, 1, 1).expand(
            -1, -1, cond_features.size(2), cond_features.size(3)
        )
        combined = torch.cat([cond_features, z_expanded], dim=1)
        return self.decoder(combined)

class Discriminator(nn.Module):
    def __init__(self, in_channels=2):  # terrain + condition
        super().__init__()
        self.model = nn.Sequential(
            nn.Conv2d(in_channels, 64, 4, 2, 1),
            nn.LeakyReLU(0.2),
            nn.Conv2d(64, 128, 4, 2, 1),
            nn.BatchNorm2d(128),
            nn.LeakyReLU(0.2),
            nn.Conv2d(128, 1, 4, 1, 1),
            nn.Sigmoid()
        )

    def forward(self, terrain, condition):
        return self.model(torch.cat([terrain, condition], dim=1))
```

### Pix2Pix (Isola et al., 2017)

A conditional GAN trained to translate between paired images. Applications to terrain:
- **Sketch → heightmap**: user draws rough mountain shapes, model fills in realistic detail
- **Biome map → texture**: biome mask → photorealistic satellite texture
- **Coarse terrain → high-res**: super-resolution for heightmaps

### CycleGAN — Unpaired Style Transfer

Learns mappings without paired training data:
- Real satellite DEM → stylized fantasy map aesthetic
- Flat noise → erosion-shaped terrain (trained on real-world DEMs)

---

## Terrain-Specific GAN Training Data

For training terrain GANs, use real-world elevation data:
- **SRTM** (Shuttle Radar Topography Mission): 30m resolution, global: https://www2.jpl.nasa.gov/srtm/
- **MERIT DEM**: hydrologically corrected: https://hydro.iis.u-tokyo.ac.jp/~yamadai/MERIT_DEM/
- **ALOS World 3D**: 30m resolution: https://www.eorc.jaxa.jp/ALOS/en/aw3d30/

---

## Diffusion Models (2022+)

Newer than GANs, **diffusion models** (DDPM, Stable Diffusion) are now state-of-the-art for image generation. They progressively denoise a random Gaussian image toward a target distribution.

Applied to terrain:
- **Terrain inpainting**: fill in missing heightmap regions
- **Text-to-terrain**: "generate a mountainous terrain with a river delta"
- **Reference-guided generation**: given a reference terrain image, generate a similar but new terrain

```
Forward process (training): x_t = sqrt(α_t)*x_0 + sqrt(1-α_t)*ε
Reverse process (inference): predict and subtract noise at each step t → t-1

Score function: ε_θ(x_t, t) → predicted noise to remove
```

ControlNet (2023) adds spatial conditioning (sketch, depth map) to diffusion models — directly applicable to terrain generation from rough sketches.

---

## References

- Gatys et al. (2015) Neural Style Transfer: https://arxiv.org/abs/1508.06576
- Goodfellow et al. (2014) GANs: https://arxiv.org/abs/1406.2661
- Isola et al. (2017) Pix2Pix: https://arxiv.org/abs/1611.07004
- CycleGAN: https://arxiv.org/abs/1703.10593
- ControlNet: https://arxiv.org/abs/2302.05543
- Wikipedia — GAN: https://en.wikipedia.org/wiki/Generative_adversarial_network
- Terrain GAN survey: https://arxiv.org/abs/2104.04634
- SRTM data: https://www2.jpl.nasa.gov/srtm/

---

## Application in Our World-Gen Pipeline

GANs and style transfer are **enhancement and finishing tools** rather than core simulators:

| Use | Layer | Details |
|---|---|---|
| **Heightmap super-resolution** | Geology | Upscale low-res tectonic output to game-resolution detail |
| **Terrain texture synthesis** | Rendering | Generate satellite-style or artistic map textures from heightmap |
| **Sketch-guided terrain** | User input | Let users sketch mountain ranges / coastlines → GAN fills in detail |
| **Style transfer for maps** | Rendering | Make output look like Tolkien hand-drawn maps, satellite imagery, or topo maps |
| **Erosion plausibility** | Geology | Discriminator trained on real DEMs can score plausibility of procedural terrain |
| **Biome texture generation** | Rendering | Generate photorealistic biome textures from biome labels |
| **Historical map aesthetic** | Rendering | Style-transfer generated maps to match historical cartography aesthetics |

**Key consideration:** ML models require training data and a GPU. This makes them less accessible than pure algorithmic approaches. However, **pre-trained models** (Stable Diffusion with ControlNet, Pix2Pix trained on DEMs) could be shipped as optional enhancement modules. The simulation pipeline works without ML; ML adds visual quality and user-driven control.
