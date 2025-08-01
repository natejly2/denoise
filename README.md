# OCT Denoising with Diffusion Models

A TensorFlow-based implementation of denoising Optical Coherence Tomography (OCT) images using Denoising Diffusion Probabilistic Models (DDPMs).

_Last updated: 2025-07-31_

---

## Table of Contents

1. [Introduction](#introduction)
2. [Mathematical Intuition](#mathematical-intuition)
3. [Project Structure](#project-structure)
4. [Image Preprocessing](#image-preprocessing)
5. [Model Architecture](#model-architecture)
6. [Training Details](#training-details)
7. [Sampling and Inference](#sampling-and-inference)
8. [Dependencies](#dependencies)
9. [References](#references)

---

# Introduction

This notebook implements a denoising pipeline using diffusion models. The method progressively adds noise to an image and then trains a neural network to remove that noise. This approach has demonstrated strong performance in generative modeling and is adapted here for enhancing OCT images, which are used in medical diagnostics.

---

# Mathematical Intuition

## Forward Process (Noising)

We define a forward Markov chain that gradually adds Gaussian noise:

```math
\bar{\alpha}_t = \prod_{s=1}^{t}(1 - \beta_s)
```
```math
q(x_t | x_{t-1}) = \mathcal{N}(x_t; \sqrt{1 - \beta_t} \cdot x_{t-1}, \beta_t \cdot I)
```

By chaining the transitions:

```math
q(x_t | x_0) = \mathcal{N}(x_t; \sqrt{\bar{\alpha}_t} x_0, (1 - \bar{\alpha}_t) I)
```

Where:  
$\beta_t$ is the noise variance schedule  

<p align="center">
<img src="oct/docImgs/1.png" width="200"/>
<img src="oct/docImgs/2.png" width="200"/>
<img src="oct/docImgs/3.png" width="200"/>
<img src="oct/docImgs/4.png" width="200"/>
<img src="oct/docImgs/5.png" width="200"/>
</p>

### Timestep Embedding

Sinusoidal embeddings are used to condition the model on the current timestep

For a scalar timestep $t$, the sinusoidal embedding vector is:

```math
\text{Embed}(t) = [\sin(t \cdot \omega_1), \cos(t \cdot \omega_1), \sin(t \cdot \omega_2), \cos(t \cdot \omega_2), \dots]
```

Where each frequency $\omega_k$ is defined as:

```math
\omega_k = \frac{1}{10000^{2k/d}}
```

- $d$ is the embedding dimension.
- The embeddings alternate between sine and cosine values across dimensions.
- This gives each timestep a unique, smooth, and continuous encoding.


## Reverse Process (Denoising)

The reverse process involves learning how to denoise an image, step by step, from Gaussian noise back to the original clean sample.


We train a neural network $\epsilon_\theta(x_t, t)$ to predict the noise $\epsilon$ that was added to a clean image $x_0$ at timestep $t$.  
The training objective is to minimize the mean squared error between the true and predicted noise:

```math
\mathcal{L}_{\text{simple}} = \mathbb{E}_{x_0, \epsilon, t} \left[ \| \epsilon - \epsilon_\theta(x_t, t) \|^2 \right]
```


### Reverse Distribution

We model the reverse diffusion step as a Gaussian conditional distribution:

```math
p_\theta(x_{t-1} \mid x_t) = \mathcal{N}(x_{t-1}; \mu_\theta(x_t, t), \Sigma_\theta(x_t, t))
```

The parameters $\mu_\theta$ and $\Sigma_\theta$ are learned functions (or derived from $\epsilon_\theta$).


### From Noised to Clean Sample

From the forward process, we have:

```math
x_t = \sqrt{\bar{\alpha}_t} x_0 + \sqrt{1 - \bar{\alpha}_t} \epsilon
```

Solving for $x_0$:

```math
x_0 = \frac{1}{\sqrt{\bar{\alpha}_t}} \left( x_t - \sqrt{1 - \bar{\alpha}_t} \epsilon \right)
```

During inference, we replace the true noise $\epsilon$ with the predicted noise $\epsilon_\theta(x_t, t)$:

```math
\hat{x}_0 = \frac{1}{\sqrt{\bar{\alpha}_t}} \left( x_t - \sqrt{1 - \bar{\alpha}_t} \cdot \epsilon_\theta(x_t, t) \right)
```

This estimate $`\hat{x}_0`$ is then used to compute the posterior mean $`\mu_\theta(x_t, t)`$, which is used to sample $`x_{t-1}`$.


---

# Project Structure

- Forward diffusion process implementation
- U-Net model with residual, attention, and normalization blocks
- Training routine with noise prediction loss
- Sampling procedure to denoise from pure noise

---

# Image Preprocessing

1. Load an OCT image (`OCT.png`)
2. Normalize image pixel values to the range [-1, 1]
4. Add noise based on a pre-defined diffusion schedule

---

# Model Architecture

The model is based on a U-Net architecture with the following enhancements:

- Sinusoidal timestep embedding
- Residual blocks with dropout and embedding projection
- Self-attention blocks at selected resolutions
- Group normalization for stability
- Downsampling and upsampling layers for feature resolution control

---

# Training Details

| Parameter       | Value       |
|----------------|-------------|
| `IMG_SIZE`     | 512         |
| `PATCH_SIZE`   | 128         |
| `BATCH_SIZE`   | 2           |
| `EPOCHS`       | 500         |
| `T`            | 100         |
| `BETA_START`   | 1e-4        |
| `BETA_END`     | 6e-3        |
| `LEARNING_RATE`| 1e-4        |

The model learns to denoise images by predicting the noise component added at each diffusion step.

---

# Sampling and Inference

To generate or recover a clean image:

1. Initialize with pure Gaussian noise \(x_T\)
2. Iteratively denoise from timestep \(T\) down to 1 using the trained model
3. The final image \(x_0\) approximates the original clean image

Visualizations at different timesteps can demonstrate the denoising process.
<p align="center">

<img src="oct/docImgs/bad1.png" width="300"/>
<img src="oct/docImgs/good1.png" width="300"/>
</p>

<p align="center">

<img src="oct/docImgs/bad2.png" width="300"/>
<img src="oct/docImgs/good2.png" width="300"/>
</p>

<p align="center">
<img src="oct/docImgs/bad3.png" width="300"/>
<img src="oct/docImgs/good3.png" width="300"/>
</p>

---

# Dependencies

- TensorFlow 2.x
- NumPy
- OpenCV
- Matplotlib

---

# References

- https://github.com/hojonathanho/diffusion
- https://github.com/DeweiHu/OCT_DDPM
