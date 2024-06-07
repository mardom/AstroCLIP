# AstroCLIP

Official PyTorch implementation and pre-trained models for paper **AstroCLIP: A Cross-Modal Foundation Model for Galaxies**.

![image](assets/im_embedding.png)

AstroCLIP is a novel, cross-modal, self-supervised foundation model that creates a shared embedding space for multi-band imaging and optical spectra of galaxies. These embeddings encode meaningful physical information shared between both modalities, and can be used as the basis for competitive zero- and few-shot learning on a variety of downstream tasks, including similarity search, redshift estimation, galaxy property prediction, and morphology classification.

## Installation
The training and evaluation code requires PyTorch 2.0. Additionally, an up-to-date eventlet is required for wandb. Note that the code has only been tested with the specified versions and also expects a Linux environment. To install the AstroCLIP package and its corresponding dependencies, please follow the code below.

```bash
pip install --upgrade pip
pip install --upgrade eventlet torch lightning[extra]
pip install -e .
```
It is possible to override default storage path by changing the flag in `astroclip/env.py`

## Pretrained Models

We provide the pretrained AstroCLIP model on the Huggingface model hub for easy access. Additionally, we provide the pretrained single-modal models for galaxy images and spectra as well. Model details, checkpoints, configs and logs are below.  

<table>
  <tr>
    <th>Model Name</th>
    <th>Pretraining</th>
    <th># Params.</th>
    <th colspan="3">Download</th>
  </tr>
  <tr>
    <td>AstroCLIP</td>
    <td>CLIP</td>
    <td>370M</td>
    <td><a href="https://github.com/PolymathicAI/AstroCLIP/blob/main/configs/astroclip.yaml">ckpt</a></td>
    <td><a href="">config</a></td>
    <td><a href="https://example.com/link3">logs</a></td>
  </tr>
  <tr>
    <td>Image Encoder</td>
    <td>DINOv2</td>
    <td>302M</td>
    <td><a href="https://example.com/link1">ckpt</a></td>
    <td><a href="https://github.com/PolymathicAI/AstroCLIP/blob/main/astroclip/astrodino/config.yaml">config</a></td>
    <td><a href="https://example.com/link3">logs</a></td>
  </tr>
  <tr>
    <td>Spectrum Encoder</td>
    <td>Masked Modeling</td>
    <td>43M</td>
    <td><a href="https://example.com/link1">ckpt</a></td>
    <td><a href="https://github.com/PolymathicAI/AstroCLIP/blob/main/configs/specformer.yaml">config</a></td>
    <td><a href="https://example.com/link3">logs</a></td>
  </tr>
</table>

#### High-Level Performance Overview

Below, we include a high-level performance overview of our models on a variety of downstream tasks. This is non-exhaustive, and we refer the reader to the paper for the full details.

<table>
  <tr>
    <th>Source</th>
    <th>Model</th>
    <th>Redshift</th>
    <th>Galaxy Property (avg)</th>
    <th>Morphology (avg)</th>
  </tr>
  <tr>
    <td>Image</td>
    <td>AstroCLIP*</td>
    <td>0.79</td>
    <td>0.47</td>
    <td>0.76</td>
  </tr>
  <tr>
    <td></td>
    <td>Image Encoder*</td>
    <td>0.63</td>
    <td>0.37</td>
    <td>0.78</td>
  </tr>
  <tr>
    <td></td>
    <td>Stein, et al.</td>
    <td>0.36</td>
    <td>0.26</td>
    <td>0.76</td>
  </tr>
  <tr>
    <td></td>
    <td>ResNet18</td>
    <td>0.77</td>
    <td>0.43</td>
    <td>-</td>
  </tr>
  <tr>
    <td></td>
    <td>ZooBoot</td>
    <td>-</td>
    <td>-</td>
    <td>0.88</td>
  </tr>
  <tr>
    <td>Spectrum</td>
    <td>AstroCLIP*</td>
    <td>0.99</td>
    <td>0.63</td>
    <td>-</td>
  </tr>
  <tr>
    <td></td>
    <td>Spectrum Encoder*</td>
    <td>0.99</td>
    <td>0.64</td>
    <td>-</td>
  </tr>
  <tr>
    <td></td>
    <td>Conv+Att</td>
    <td>0.99</td>
    <td>0.60</td>
    <td>-</td>
  </tr>
  <tr>
    <td>Photometry</td>
    <td>MLP</td>
    <td>0.68</td>
    <td>0.42</td>
    <td>-</td>
  </tr>
  <tr>
</table>

We report R-squared metrics on redshift and galaxy property estimation and accuracy on galaxy morphology classification (averaged across all labels).Our models are marked with an asterisk (*). The AstroCLIP/Encoder/Stein et al. models are evaluated with zero-shot learning on the embeddings.

## Data Access

The AstroCLIP model is trained on the cross-matched sample containing optical spectra from the [Dark Energy Spectroscopic Instrument (DESI)](https://www.desi.lbl.gov/) Early Data Release (EDR) and multi-band images (g,r,z) from the [DESI Legacy Survey](https://www.legacysurvey.org/) prepared by [Stein, et al. (2022)](https://github.com/georgestein/ssl-legacysurvey/tree/main). We provide the dataset as a HuggingFace dataset, which can be accessed directly using

```python
from datasets import load_dataset

# This downloads about 60 GB of data
dset = load_dataset('astroclip/datasets/legacy_survey.py')
```

For reproducibility, we include the scripts to generate the cross-matched datasets [here]().

### Image Pretraining Dataset

![image](assets/decals.png)

While the AstroCLIP and Spectrum Encoder models are trained on the image-spectrum dataset, we pretrain the galaxy image model separately on full Stein, et al. (2022) image dataset, which consists of 76M galaxy images. This dataset can be accessed using this globus endpoint:

https://app.globus.org/file-manager?origin_id=9fb0fc0e-e760-11ec-9bd2-2d2219dcc1fa&origin_path=%2F

The directory is organized into south and north surveys, where each survey is split into chunks of 1,000,000 galaxies (sorted by decreasing z-band flux) and saved in hdf5 format. For more details, see [here](https://github.com/georgestein/ssl-legacysurvey/tree/main).


## Pretraining

AstroCLIP is trained using a two-step process. First, we pre-train a single-modal galaxy image encoder and a single-modal galaxy spectrum encoder separately. Then, we CLIP align these two encoders on a paired image-spectrum dataset.

### Image Pretraining - DINOv2 ViT:
AstroCLIP uses a Vision Transformer (ViT) to encode galaxy images. Pretraining is performed using the [DINOv2](https://github.com/facebookresearch/dinov2/) package, which combines self-distillation, masked-modeling, and contrastive objectives. Overall, we use largely the same training regime, however we modify some of the contrastive augmentations to suit an astrophysics context.

Model training can be launched with the following command:
```
image_trainer -c astroclip/astrodino/config.yaml
```
We train the model using 20 A100 GPUs (on 5 nodes) for 250k steps which takes roughly 46 hours. 

### Spectrum Pretraining - Masked Modelling Transformer:
AstroCLIP uses a 1D Transformer to encode galaxy spectra. Pretraining is performed using a masked-modeling objective, whereby the 1D spectrum is split into contiguous, overlapping patches. 

Model training can be launched with the following command:
```
spectrum_trainer fit -c config/specformer.yaml
```
We train the model using 4 A100 GPUs (on 1 node) for 30k steps which takes roughly 12 hours.

### CLIP Alignment:

Once pretrained, we align the image and spectrum encoder using cross-attention projection heads. Model training can be launched with the following command:
```
spectrum_trainer fit -c config/astroclip.yaml
```
We train the model using 4 A100 GPUs (on 1 node) for 15k steps which takes roughly 12 hours.

## Downstream Tasks

TODO

## Acknowledgements
This reposity uses datasets and contrastive augmentations from [Stein, et al. (2022)](https://github.com/georgestein/ssl-legacysurvey/tree/main). The image pretraining is built on top of the [DINOv2 pretraining framework](https://github.com/facebookresearch/dinov2/).

## License
AstroCLIP code and model weights are released under the MIT license. See [LICENSE](https://github.com/PolymathicAI/AstroCLIP/blob/main/LICENSE) for additional details.

## Citations
TODO