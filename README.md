# Dense Video Captioning with Bi-modal Transformer

Reference : https://v-iashin.github.io/bmt
## Summary

Dense video captioning aims to localize and describe important events in untrimmed videos. Existing methods mainly tackle this task by exploiting the visual information alone, while completely neglecting the audio track. 

To this end, we present *Bi-modal Transformer with Proposal Generator* (BMT), which efficiently utilizes audio and visual input sequences to select events in a video and, then, use these clips to generate a textual description.

<img src="https://github.com/v-iashin/v-iashin.github.io/raw/master/images/bmt/bi_modal_transformer.svg" alt="Bi-Modal Transformer with Proposal Generator" width="900">

Audio and visual features are encoded with [*VGGish*](https://github.com/tensorflow/models/tree/0b3a8abf095cb8866ca74c2e118c1894c0e6f947/research/audioset/vggish) and [*I3D*](https://github.com/hassony2/kinetics_i3d_pytorch/tree/51240948f9ae92808c390e7217041d6fd89414e9) while caption tokens with [*GloVe*](https://torchtext.readthedocs.io/en/latest/vocab.html#glove). First, VGGish and I3D features are passed through the stack of *N* bi-modal encoder layers where audio and visual sequences are encoded to, what we call, audio-attended visual and video-attended audio features. These features are passed to the bi-modal multi-headed proposal generator, which generates a set of proposals using information from both modalities. 

Then, the input features are trimmed according to the proposed segments and encoded in the bi-modal encoder again. The stack of *N* bi-modal decoder layers inputs both: a) GloVe embeddings of the previously generated caption sequence, b) the internal representation from the last layer of the encoder for both modalities. The decoder produces its internal representation which is, then, used in the generator model the distribution over the vocabulary for the caption next word.

<!-- <img src="https://github.com/v-iashin/v-iashin.github.io/raw/master/images/bmt/proposal_generator_compressed.svg" alt="Bi-Modal Transformer with Proposal Generator" height="400"> -->

## Getting Started
Clone the repository. Mind the `--recursive` flag to make sure `submodules` are also cloned (evaluation scripts for Python 3).
```bash
git clone --recursive https://github.com/saikaran-ss20/VideoCaptioning.git
```

Download features (I3D and VGGish) and word embeddings (GloVe). The script will download them (~10 GB) and unpack into `./data` and `./.vector_cache` folders. *Make sure to run it while being in BMT folder*
```bash
bash ./download_data.sh
```

Set up a `conda` environment
```bash
conda env create -f ./conda_env.yml
conda activate bmt
# install spacy language model. Make sure you activated the conda environment
python -m spacy download en
```

## Train

We train our model in two staged: training of the captioning module on ground truth proposals and training of the proposal generator using the pre-trained encoder from the captioning module.

- *Train the captioning module*. You may also download the pre-trained model [best_cap_model.pt](https://a3s.fi/swift/v1/AUTH_a235c0f452d648828f745589cde1219a/bmt/best_cap_model.pt) (`md5 hash 7b4d48cd77ec49a027a4a1abc6867ee7`).
```bash
python main.py \
    --procedure train_cap \
    --B 32
    --device_ids 0
```

- *Train proposal generation module*. You may also download the pre-trained model [best_prop_model.pt](https://a3s.fi/swift/v1/AUTH_a235c0f452d648828f745589cde1219a/bmt/best_prop_model.pt) (`md5 hash 5f8b20826b09eadd41b7a5be662c198b`)
```bash
python main.py \
    --procedure train_prop \
    --pretrained_cap_model_path /your_exp_path/best_cap_model.pt \
    --B 16
    --device_ids 0
```

## Evaluate

Since a part of videos in ActivityNet Captions became unavailable over the time, we could only obtain ~91 % of videos in the dataset (see `./data/available_mp4.txt` for ids). To this end, we evaluate the performance of our model against ~91 % of the validation videos. We provide the validation sets without such videos in `./data/val_*_no_missings.json`. Please see `Experiments` and `Supplementary Material` sections for details and performance of other models on the same validation sets.

- *Ground truth proposals*. The performance of the captioning module on ground truth segments might be obtained from the file with pre-trained captioning module. You may also want to use the [official evaluation script](https://github.com/ranjaykrishna/densevid_eval/tree/9d4045aced3d827834a5d2da3c9f0692e3f33c1c) with `./data/val_*_no_missings.json` as references (`-r` argument).
```python
import torch
cap_model_cpt = torch.load('./path_to_pre_trained_model/best_cap_model.pt', map_location='cpu')
print(cap_model_cpt['val_1_metrics'])
print(cap_model_cpt['val_2_metrics'])
# To obtain the final results, average values in both dicts
```

- *Learned proposals*. Create a file with captions for every proposal provided in `--prop_pred_path` using the captioning model specified in `--pretrained_cap_model_path`. The script will automatically evaluate it againts both ground truth validation sets. Alternatively, use the predictions `prop_results_val_1_e17_maxprop100.json` in `./results` and [official evaluation script](https://github.com/ranjaykrishna/densevid_eval/tree/9d4045aced3d827834a5d2da3c9f0692e3f33c1c) with `./data/val_*_no_missings.json` as references (`-r` argument).
```bash
python main.py \
    --procedure evaluate \
    --pretrained_cap_model_path /path_to_best_cap_model.pt \
    --prop_pred_path /path_to_generated_json_file \
    --device_ids 0
```


## Details on Feature Extraction
Check out FeatureExtraction folder for extraction of i3d and vggish features of videos.

## Results

|                                    Model | Params (Mill) | BLEU@3 | BLEU@4 | METEOR |
|-----------------------------------------:|--------------:|-------:|-------:|-------:|
|                                      BMT |            51 |   4.63 |   1.99 |  10.90 |


## Single Video Prediction

*Disclaimer: we do not guarantee perfect results nor recommend you to use it in production. Sometimes captions are redundant, unnatural, and rediculous.*

Start with feature extraction using "Feature Extraction". Extract I3D features
```bash
# run this from Feature Extraction folder
conda activate i3d
python main.py \
    --feature_type i3d \
    --on_extraction save_numpy \
    --device_ids 0 \
    --extraction_fps 25 \
    --stack_size 24 \
    --step_size 24 \
    --video_paths ../BMT/test/women_long_jump.mp4
```

Extract VGGish features
```bash
# run this from Feature Extraction folder
conda activate vggish
python main.py \
    --feature_type vggish \
    --on_extraction save_numpy \
    --device_ids 0 \
    --video_paths ../BMT/test/women_long_jump.mp4
```

Run the inference
```bash
conda activate bmt
python ./sample/single_video_prediction.py \
    --prop_generator_model_path best_prop_model.pt \
    --pretrained_cap_model_path best_cap_model.pt \
    --vggish_features_path  test/women_long_jump_vggish.npy \
    --rgb_features_path  test/women_long_jump_rgb.npy \
    --flow_features_path  test/women_long_jump_flow.npy \
    --duration_in_secs 35.155 \
    --device_id 0 \
    --max_prop_per_vid 100 \
    --nms_tiou_thresh 0.4
```

Expected output
```
[
  {'start': 0.0, 'end': 4.9, 'sentence': 'We see the closing title screen'}, 
  {'start': 2.7, 'end': 29.0, 'sentence': 'A woman is seen running down a track and down a track while others watch on the sides'}, 
  {'start': 19.6, 'end': 33.3, 'sentence': 'The man runs down the track and jumps into a sand pit'}, 
  {'start': 0.0, 'end': 13.0, 'sentence': 'A man is seen running down a track and leads into a large group of people running around a track'}, 
  {'start': 0.9, 'end': 2.5, 'sentence': 'We see a title screen'}, 
  {'start': 0.0, 'end': 1.6, 'sentence': 'A man is seen sitting on a table with a white words on the screen'}, 
  {'start': 30.0, 'end': 35.2, 'sentence': 'The man runs down the track and lands on the sand'}
]
```

Note that in our research we avoided non-maximum suppression for computational efficiency and to allow the event prediction to be dense. Feel free to play with `--nms_tiou_thresh` parameter: for example, try to make it `0.4` as in the provided example. 

The sample video credits: [Women's long jump historical World record in 1978](https://www.youtube.com/watch?v=nynA-Gmh2r8)

## Citation
Please, use this bibtex if you would like to cite our work
```
@misc{BMT_Iashin_2020,
  title={A Better Use of Audio-Visual Cues: Dense Video Captioning with Bi-modal Transformer},
  author={Vladimir Iashin and Esa Rahtu},
  year={2020},
  eprint={2005.08271},
  archivePrefix={arXiv},
  primaryClass={cs.CV}
}
```
```
@InProceedings{MDVC_Iashin_2020,
  author = {Iashin, Vladimir and Rahtu, Esa},
  title = {Multi-modal Dense Video Captioning},
  booktitle = {Workshop on Multimodal Learning (CVPR Workshop)},
  year = {2020}
}
```
