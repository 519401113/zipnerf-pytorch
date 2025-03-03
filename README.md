# ZipNeRF

An unofficial pytorch implementation of 
"Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields" 
[https://arxiv.org/abs/2304.06706](https://arxiv.org/abs/2304.06706).
This work is based on [multinerf](https://github.com/google-research/multinerf), so features in refnerf,rawnerf,mipnerf360 are also available.


## Results
Current results:  

https://user-images.githubusercontent.com/83005605/234030861-ee649921-2bd2-4ebd-8203-1192acad3bc3.mp4


|             | bicycle | garden | stump | room  | counter | kitchen | bonsai |
|:-----------:|:-------:|:------:|:-----:|:-----:|:-------:|:-------:|:------:|
|    PSNR     |  24.45  | 26.18  | 26.46 | 31.79 |  28.56  |  31.35  | 33.47  |
|    SSIM     |  0.685  | 0.752  | 0.762 | 0.914 |  0.896  |  0.909  | 0.937  |

The video is rendered at a downsample factor of 8 while the nerf model is trained in full resolution. Training speed is about 1.5x slower than paper(1.5 hours on 8 A6000).

This project is work-in-progress, and any advice will be appreciated.
## Install

```
# Clone the repo.
git clone https://https://github.com/SuLvXiangXin/zipnerf-pytorch.git
cd zipnerf-pytorch

# Make a conda environment.
conda create --name zipnerf python=3.9
conda activate zipnerf

# Install requirements.
pip install -r requirements.txt

# Install other extensions
pip install ./gridencoder

# Install a specific cuda version of torch_scatter 
# see more detail at https://github.com/rusty1s/pytorch_scatter
CUDA=cu117
pip install torch-scatter -f https://data.pyg.org/whl/torch-2.0.0+${CUDA}.html
```

## Dataset
Only [mipnerf360 data](http://storage.googleapis.com/gresearch/refraw360/360_v2.zip) is tested

```
mkdir data
cd data
wget http://storage.googleapis.com/gresearch/refraw360/360_v2.zip
unzip 360_v2.zip
```

## Train
```
# Configure your training (DDP? fp16? ...)
# see https://huggingface.co/docs/accelerate/index for details
accelerate config

# Where your data is 
DATA_DIR=data/360_v2/bicycle
EXP_NAME=360_v2/bicycle

# Experiment will be conducted under "exp/${EXP_NAME}" folder
# "--gin_configs=configs/360.gin" can be seen as a default config 
# and you can add specific config useing --gin_bindings="..." 
accelerate launch train.py \
    --gin_configs=configs/360.gin \
    --gin_bindings="Config.data_dir = '${DATA_DIR}'" \
    --gin_bindings="Config.exp_name = '${EXP_NAME}'" \
    --gin_bindings="Config.factor = 0"

# or you can also run without accelerate (without DDP)
CUDA_VISIBLE_DEVICES=0 python train.py \
    --gin_configs=configs/360.gin \
    --gin_bindings="Config.data_dir = '${DATA_DIR}'" \
    --gin_bindings="Config.exp_name = '${EXP_NAME}'" \
      --gin_bindings="Config.factor = 0" 

# alternative you can use an example training script 
bash script/train_360.sh

# metric, render image, etc can be viewed through tensorboard
tensorboard --logdir "exp/${EXP_NAME}"

```

### Render
Rendering results can be found in the directory `exp/${EXP_NAME}/render`
```
accelerate launch render.py \
    --gin_configs=configs/360.gin \
    --gin_bindings="Config.data_dir = '${DATA_DIR}'" \
    --gin_bindings="Config.exp_name = '${EXP_NAME}'" \
    --gin_bindings="Config.render_path = True" \
    --gin_bindings="Config.render_path_frames = 480" \
    --gin_bindings="Config.render_video_fps = 60" \
    --gin_bindings="Config.factor = 0"  

# alternative you can use an example rendering script 
bash script/render_360.sh
```
## Evaluate
Evaluating results can be found in the directory `exp/${EXP_NAME}/test_preds`
```
# using the same exp_name as in training
accelerate launch eval.py \
    --gin_configs=configs/360.gin \
    --gin_bindings="Config.data_dir = '${DATA_DIR}'" \
    --gin_bindings="Config.exp_name = '${EXP_NAME}'" \
    --gin_bindings="Config.factor = 0"


# alternative you can use an example evaluating script 
bash script/eval_360.sh
```
## OutOfMemory
you can decrease the total batch size by 
adding e.g.  `--gin_bindings="Config.batch_size = 8192" `, 
or decrease the test chunk size by adding e.g.  `--gin_bindings="Config.render_chunk_size = 8192" `,
or use more GPU by configure `accelerate config` .


## Preparing custom data
More details can be found at https://github.com/google-research/multinerf
```
DATA_DIR=my_dataset_dir
bash scripts/local_colmap_and_resize.sh ${DATA_DIR}
```

## Citation
```
@misc{barron2023zipnerf,
      title={Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields}, 
      author={Jonathan T. Barron and Ben Mildenhall and Dor Verbin and Pratul P. Srinivasan and Peter Hedman},
      year={2023},
      eprint={2304.06706},
      archivePrefix={arXiv},
      primaryClass={cs.CV}
}

@misc{multinerf2022,
      title={{MultiNeRF}: {A} {Code} {Release} for {Mip-NeRF} 360, {Ref-NeRF}, and {RawNeRF}},
      author={Ben Mildenhall and Dor Verbin and Pratul P. Srinivasan and Peter Hedman and Ricardo Martin-Brualla and Jonathan T. Barron},
      year={2022},
      url={https://github.com/google-research/multinerf},
}

@Misc{accelerate,
  title =        {Accelerate: Training and inference at scale made simple, efficient and adaptable.},
  author =       {Sylvain Gugger, Lysandre Debut, Thomas Wolf, Philipp Schmid, Zachary Mueller, Sourab Mangrulkar},
  howpublished = {\url{https://github.com/huggingface/accelerate}},
  year =         {2022}
}

@misc{torch-ngp,
    Author = {Jiaxiang Tang},
    Year = {2022},
    Note = {https://github.com/ashawkey/torch-ngp},
    Title = {Torch-ngp: a PyTorch implementation of instant-ngp}
}
```

## Acknowledgements
This work is based on my another repo https://github.com/SuLvXiangXin/multinerf-pytorch, 
which is basically a pytorch translation from [multinerf](https://github.com/google-research/multinerf)

- Thanks to [multinerf](https://github.com/google-research/multinerf) for amazing multinerf(MipNeRF360,RefNeRF,RawNeRF) implementation
- Thanks to [accelerate](https://github.com/huggingface/accelerate) for distributed training
- Thanks to [torch-ngp](https://github.com/ashawkey/torch-ngp) for super useful hashencoder
- Thanks to [Yurui Chen](https://github.com/519401113) for discussing the details of the paper.
