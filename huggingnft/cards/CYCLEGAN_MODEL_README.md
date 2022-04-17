---
tags:
- huggan
- gan
# See a list of available tags here:
# https://github.com/huggingface/hub-docs/blob/main/js/src/lib/interfaces/Types.ts#L12
# task: unconditional-image-generation or conditional-image-generation or image-to-image
license: mit
---

# CycleGAN for unpaired image-to-image translation. 

## Model description  

CycleGAN for unpaired image-to-image translation.   
Given two image domains A and B, the following components are trained end2end to translate between such domains:   
- A generator A to B, named G_AB conditioned on an image from A   
- A generator B to A, named G_BA conditioned on an image from B   
- A domain classifier D_A, associated with G_AB   
- A domain classifier D_B, associated with G_BA    
At inference time, G_AB or G_BA are relevant to translate images, respectively A to B or  B to A.  
In the general setting, this technique provides style transfer functionalities between the selected image domains A and B.   
This allows to obtain a generated translation by G_AB, of an image from domain A that resembles the distribution of the images from domain B, and viceversa for the generator G_BA.  
Under these framework, these aspects have been used to perform style transfer between NFT collections.   
A collection is selected as domain A, another one as domain B and the CycleGAN provides forward and backward translation between A and B.   
This has showed to allows high quality translation even in absence of paired sample-ground-truth data.  
In particular, the model performs well with stationary backgrounds (no drastic texture changes in the appearance of backgrounds) as it is capable of recognizing the attributes of each of the elements of an NFT collections.  
An attribute can be a variation in type of dressed fashion items such as sunglasses, earrings, clothes and also face or body attributes with respect to a common template model of the given NFT collection).    


## Intended uses & limitations

#### How to use

```python
import torch
from PIL import Image
from huggan.pytorch.cyclegan.modeling_cyclegan import GeneratorResNet
from torchvision import transforms as T
from torchvision.transforms import Compose, Resize, ToTensor, Normalize
from torchvision.utils import make_grid
from huggingface_hub import hf_hub_download, file_download
from accelerate import Accelerator
import json

def load_lightweight_model(model_name):
    file_path = file_download.hf_hub_download(
        repo_id=model_name,
        filename="config.json"
    )
    config = json.loads(open(file_path).read())
    organization_name, name = model_name.split("/")
    model = Trainer(**config, organization_name=organization_name, name=name)
    model.load(use_cpu=True)
    model.accelerator = Accelerator()
    return model
def get_concat_h(im1, im2):
    dst = Image.new('RGB', (im1.width + im2.width, im1.height))
    dst.paste(im1, (0, 0))
    dst.paste(im2, (im1.width, 0))
    return dst    


n_channels = 3
image_size = 256
input_shape = (image_size, image_size)

transform = Compose([
    T.ToPILImage(),
    T.Resize(input_shape),
    ToTensor(),
    Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5)),
])

# load the translation model from source to target images: source will be generated by a separate Lightweight GAN, w
# while the target images are the result of the translation applied by the GeneratorResnet to the generated source images.
# Hence, given the source domain A and target domain B,
#  B = Translator(GAN(A))
translator = GeneratorResNet.from_pretrained(f'huggingnft/{model_name}',
                                                 input_shape=(n_channels, image_size, image_size),
                                                 num_residual_blocks=9)

# sample noise that is used to generate source images by the 
z = torch.randn(nrows, 100, 1, 1)
# load the GAN generator of source images that will be translated by the translation model
model = load_lightweight_model(f"huggingnft/{model_name.split('__2__')[0]}")
collectionA = model.generate_app(
        num=timestamped_filename(),
        nrow=nrows,
        checkpoint=-1,
        types="default"
    )[1]
# resize to translator model input shape
resize = T.Resize((256, 256))
input = resize(collectionA)

# translate the resized collectionA to collectionB
collectionB = translator(input)

out_transform = T.ToPILImage()
results = []
for collA_image, collB_image in zip(input, collectionB):
    results.append(
        get_concat_h(out_transform(make_grid(collA_image, nrow=1, normalize=True)), out_transform(make_grid(collB_image, nrow=1, normalize=True)))
    )
```



#### Limitations and bias

Translation between collections provides exceptional output images in the case of NFT collections that portray subjects in the same way.  
If the backgrounds vary too much within either of the collections, performance degrades or many more training iterations re required to achieve acceptable results.

## Training data


The CycleGAN model is trained on an unpaired dataset of samples from two selected NFT collections: colle tionA and collectionB.   
To this end, two collections are loaded by means of the function load_dataset in the huggingface library, as follows.
A list of all available collections is available at [huggingNFT](https://huggingface.co/huggingnft)
```python
from datasets import load_dataset

collectionA = load_dataset("huggingnft/COLLECTION_A")
collectionB = load_dataset("huggingnft/COLLECTION_B")
```



## Training procedure
#### Preprocessing
The following transformations are applied to each input sample of collectionA and collectionB.   
The input size is fixed to RGB images of height, width = 256, 256    
```python
n_channels = 3
image_size = 256
input_shape = (image_size, image_size)

transform = Compose([
    T.ToPILImage(),
    T.Resize(input_shape),
    ToTensor(),
    Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5)),
])
```

#### Hardware  
The configuration has been tested on single GPU setup on a RTX5000 and A5000, as well as multi-gpu single-rank distributed setups composed of 2 of the mentioned GPUs.

#### Hyperparameters
The following configuration has been kept fixed for all translation models:   
- learning rate 0.0002   
- number of epochs 200
- learning rate decay activation at epoch 80
- number of residual blocks of the cyclegan 9
- cycle loss weight 10.0
- identity loss weight 5.0
- optimizer ADAM with beta1 0.5 and beta2 0.999
- batch size 8
- NO mixed precision training

## Eval results


#### Training reports

[Cryptopunks to boreapeyachtclub](https://wandb.ai/chris1nexus/experiments--experiments_cyclegan_punk_to_apes_HQ--0/reports/CycleGAN-training-report--VmlldzoxODUxNzQz?accessToken=vueurpbhd2i8n347j880yakggs0sqdf7u0hpz3bpfsbrxcmk1jk4obg18f6wfk9w)


[Boreapeyachtclub to mutant-ape-yacht-club](https://wandb.ai/chris1nexus/experiments--my_paperspace_boredapeyachtclub__2__mutant-ape-yacht-club--11/reports/CycleGAN-training-report--VmlldzoxODUxNzg4?accessToken=jpyviwn7kdf5216ycrthwp6l8t3heb0lt8djt7dz12guu64qnpdh3ekecfcnoahu)


#### ## Generated Images

In the provided images, row0 and row2 represent real images from the respective collections.  
Row1 is the translation of the immediate above images in row0 by means of the G_AB translation model.  
Row3 is the translation of the immediate above images in row2 by means of the G_BA translation model.  

 Visualization over the training iterations for [boreapeyachtclub to mutant-ape-yacht-club](https://wandb.ai/chris1nexus/experiments--my_paperspace_boredapeyachtclub__2__mutant-ape-yacht-club--11/reports/Shared-panel-22-04-15-08-04-99--VmlldzoxODQ0MDI3?accessToken=45m3kxex5m3rpev3s6vmrv69k3u9p9uxcsp2k90wvbxwxzlqbqjqlnmgpl9265c0) 

 Visualization over the training iterations for [Cryptopunks to boreapeyachtclub](https://wandb.ai/chris1nexus/experiments--experiments_cyclegan_punk_to_apes_HQ--0/reports/Shared-panel-22-04-17-11-04-83--VmlldzoxODUxNjk5?accessToken=o25si6nflp2xst649vt6ayt56bnb95mxmngt1ieso091j2oazmqnwaf4h78vc2tu) 


### References

@misc{https://doi.org/10.48550/arxiv.1703.10593,
  doi = {10.48550/ARXIV.1703.10593},
  
  url = {https://arxiv.org/abs/1703.10593},
  
  author = {Zhu, Jun-Yan and Park, Taesung and Isola, Phillip and Efros, Alexei A.},
  
  keywords = {Computer Vision and Pattern Recognition (cs.CV), FOS: Computer and information sciences, FOS: Computer and information sciences},
  
  title = {Unpaired Image-to-Image Translation using Cycle-Consistent Adversarial Networks},
  
  publisher = {arXiv},
  
  year = {2017},
  
  copyright = {arXiv.org perpetual, non-exclusive license}
}

### BibTeX entry and citation info

```bibtex
@InProceedings{huggingnft,
    author={Aleksey Korshuk, Christian Cancedda}
    year=2022
}
```