---
title: "Dimensionality Reduction: From PCA to Autoencoders"
date: 2025-03-21T04:00:00-07:00
draft: false
math: true
---

Dimensionality reduction is a cornerstone of data analysis, enabling us to simplify high-dimensional datasets while preserving essential information. This article explores three unsupervised methods—Principal Component Analysis (PCA), linear dimensionality reduction, and autoencoders—each offering unique insights into data compression. We'll conclude with their practical utility in scenarios with limited labeled data.

## Principal Component Analysis (PCA)

PCA is a statistical technique that transforms a dataset into a new coordinate system where the axes (principal components) are orthogonal and ordered by the variance they capture. Given a dataset $X$ with $n$ observations and $p$ features, PCA seeks to project $X$ onto a lower-dimensional space.

The process starts by computing the covariance matrix of $X$, followed by its eigendecomposition. The eigenvectors represent the principal components, and the eigenvalues indicate the variance along each direction. We select the top $k$ eigenvectors to reduce the dimensionality from $p$ to $k$.

Mathematically, for a centered dataset $X$, PCA solves:

$$
\text{Cov}(X) = \frac{1}{n-1} X^T X
$$

The projection of $X$ onto the top $k$ components is:

$$
Z = X W_k
$$

where $W_k$ is the matrix of the $k$ largest eigenvectors.

### PCA Distribution Plot
Imagine a 2D scatter plot of a dataset with two features, say height and weight. After PCA, the first principal component (PC1) aligns with the direction of maximum variance, and the second (PC2) is orthogonal. Below is the plot:

![PCA Scatter Plot](/images/plots.png)

*Caption: A typical PCA distribution. Original data (left) is projected onto PC1 and PC2 (right), showing variance concentration along PC1.*

## Linear Dimensionality Reduction

Linear dimensionality reduction generalizes PCA by applying a linear transformation $f(X) = XW$, where $W$ is a matrix mapping from $p$ dimensions to $k < p$. PCA is a special case, optimizing for variance preservation, but other methods (e.g., Factor Analysis) might prioritize different criteria.

A key insight is that stacking multiple linear layers doesn't enhance expressiveness. If $f_1(X) = XW_1$ and $f_2(X) = f_1(X)W_2$, the composition is:

$$
f_2(f_1(X)) = (XW_1)W_2 = X(W_1W_2) = XW
$$

Since $W_1W_2$ is still a single linear transformation, additional layers don't capture nonlinear patterns—unlike neural networks, which we'll explore next.

## Autoencoders: Nonlinear Dimensionality Reduction

Autoencoders are neural networks designed to learn a compressed representation of data. They consist of an encoder, which maps input $X$ to a latent space $Z$, and a decoder, which reconstructs $X$ from $Z$. The goal is to minimize reconstruction error:

$$
L = \|X - \hat{X}\|^2
$$

where $\hat{X}$ is the decoder's output.

The encoder might look like:

$$
Z = \sigma(WX + b)
$$

and the decoder:

$$
\hat{X} = \sigma'(W'Z + b')
$$

Here, $\sigma$ and $\sigma'$ are nonlinear activation functions (e.g., ReLU, sigmoid), allowing autoencoders to capture nonlinear relationships—unlike PCA or linear methods.

### Connection to U-Net
Autoencoders share conceptual similarities with U-Net, a convolutional network used in image segmentation. U-Net's encoder-decoder structure with skip connections resembles a deep autoencoder, though U-Net is supervised and task-specific. Autoencoders, being unsupervised, focus solely on reconstruction.

### Neural Encoder Diagram
A multi-layer encoder might have several fully connected layers, narrowing to a bottleneck (latent space). Here's the diagram:

![Neural Network Diagram](/images/autoencoder.png)

*Caption: A neural encoder with input layer (p=5), a hidden layer (4 nodes), and latent space (k=2). The decoder mirrors this structure.*

## Unsupervised Nature

All three methods—PCA, linear reduction, and autoencoders—are unsupervised. They don't require labeled data, relying instead on the intrinsic structure of $X$ (e.g., variance for PCA, reconstruction for autoencoders). This makes them versatile for exploratory analysis or preprocessing.

## Conclusion: Practical Utility

These techniques shine when data is scarce and labels are sparser still. In such cases, a hybrid strategy works well:
1. Train an autoencoder on the entire dataset (labeled and unlabeled) to learn a latent space $Z$.
2. Use $Z$ as input to a supervised model, trained only on the labeled subset.

PCA and linear methods are computationally efficient for small-to-medium datasets, while autoencoders excel with complex, nonlinear data (e.g., images). By leveraging unsupervised pretraining, we maximize the use of all available data, enhancing performance on downstream tasks.

---