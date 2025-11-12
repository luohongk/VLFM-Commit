# VLFM-Commit

## 适配CUDA11.8、habitat-sim0.2.4版本的VLFM，并给出详细的代码理解注释

<p align="center">
  <img src="docs/teaser_v1.jpg" width="700">
  <h1 align="center">VLFM: Vision-Language Frontier Maps for Zero-Shot Semantic Navigation</h1>
  <h3 align="center">
    <a href="http://naoki.io/">Naoki Yokoyama</a>, <a href="https://faculty.cc.gatech.edu/~sha9/">Sehoon Ha</a>, <a href="https://faculty.cc.gatech.edu/~dbatra/">Dhruv Batra</a>, <a href="https://www.robo.guru/about.html">Jiuguang Wang</a>, <a href="https://bucherb.github.io">Bernadette Bucher</a>
  </h3>
  <p align="center">
    <a href="http://naoki.io/portfolio/vlfm.html">Project Website</a> , <a href="https://arxiv.org/abs/2312.03275">Paper (arXiv)</a>
  </p>
  <p align="center">
    <a href="https://github.com/bdaiinstitute/vlfm">
      <img src="https://img.shields.io/badge/License-MIT-yellow.svg" />
    </a>
    <a href="https://www.python.org/">
      <img src="https://img.shields.io/badge/built%20with-Python3-red.svg" />
    </a>
    <a href="https://github.com/jiuguangw/Agenoria/actions">
      <img src="https://github.com/bdaiinstitute/vlfm/actions/workflows/test.yml/badge.svg">
    </a>
    <a href="https://github.com/psf/black">
      <img src="https://img.shields.io/badge/code%20style-black-000000.svg">
    </a>
    <a href="https://github.com/astral-sh/ruff">
      <img src="https://img.shields.io/endpoint?url=https://raw.githubusercontent.com/charliermarsh/ruff/main/assets/badge/v2.json">
    </a>
    <a href="https://github.com/python/mypy">
      <img src="http://www.mypy-lang.org/static/mypy_badge.svg">
    </a>
  </p>
</p>

## ✨ Overview

Understanding how humans leverage semantic knowledge to navigate unfamiliar environments and decide where to explore next is pivotal for developing robots capable of human-like search behaviors. We introduce a zero-shot navigation approach, Vision-Language Frontier Maps (VLFM), which is inspired by human reasoning and designed to navigate towards unseen semantic objects in novel environments. VLFM builds occupancy maps from depth observations to identify frontiers, and leverages RGB observations and a pre-trained vision-language model to generate a language-grounded value map. VLFM then uses this map to identify the most promising frontier to explore for finding an instance of a given target object category. We evaluate VLFM in photo-realistic environments from the Gibson, Habitat-Matterport 3D (HM3D), and Matterport 3D (MP3D) datasets within the Habitat simulator. Remarkably, VLFM achieves state-of-the-art results on all three datasets as measured by success weighted by path length (SPL) for the Object Goal Navigation task. Furthermore, we show that VLFM's zero-shot nature enables it to be readily deployed on real-world robots such as the Boston Dynamics Spot mobile manipulation platform. We deploy VLFM on Spot and demonstrate its capability to efficiently navigate to target objects within an office building in the real world, without any prior knowledge of the environment. The accomplishments of VLFM underscore the promising potential of vision-language models in advancing the field of semantic navigation.

## 🛠 Installation

### Getting Started

Create the conda environment:

```bash
conda create -n vlfm python=3.9 cmake=3.14.0
conda activate vlfm
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
sudo apt update
sudo apt install build-essential
sudo apt install gcc g++
pip install git+https://github.com/IDEA-Research/GroundingDINO.git@34ef00dcdf5cadb84c21b59db4fc42a4d4c75047 salesforce-lavis==1.0.2

#cd VLFM主文件夹

#克隆yolov7
git clone git@github.com:WongKinYiu/yolov7.git

#克隆GroundingDINO
git clone https://github.com/IDEA-Research/GroundingDINO.git


# 带显示器的机器
conda install habitat-sim=0.2.4  withbullet -c conda-forge -c aihabitat

# 无显示器的机器（如集群）
# conda install habitat-sim=0.2.4 withbullet headless -c conda-forge -c aihabitat

```

## 🎯 Downloading the HM3D dataset

### Matterport

First, set the following variables during installation (don't need to put in .bashrc):

```bash
# 设置API令牌
MATTERPORT_TOKEN_ID=a5e78a71xxxxxx
MATTERPORT_TOKEN_SECRET=676cd40457b6b0fxxxx

DATA_DIR=</path/to/vlfm/data>
```

### Clone and install habitat-lab, then download datasets

*Ensure that the correct conda environment is activated!!*

```bash
# 下载HM3D训练集
python -m habitat_sim.utils.datasets_download \\
  --username $MATTERPORT_TOKEN_ID --password $MATTERPORT_TOKEN_SECRET \\
  --uids hm3d_train_v0.2 \\
  --data-path $DATA_DIR

# 下载HM3D验证集
python -m habitat_sim.utils.datasets_download \\
  --username $MATTERPORT_TOKEN_ID --password $MATTERPORT_TOKEN_SECRET \\
  --uids hm3d_val_v0.2 \\
  --data-path $DATA_DIR

wget <https://dl.fbaipublicfiles.com/habitat/data/datasets/objectnav/hm3d/v1/objectnav_hm3d_v1.zip> &&
unzip objectnav_hm3d_v1.zip &&
mkdir -p $DATA_DIR/datasets/objectnav/hm3d  &&
mv objectnav_hm3d_v1 $DATA_DIR/datasets/objectnav/hm3d/v1 &&
rm objectnav_hm3d_v1.zip
```

## :weight_lifting: Downloading weights for various models

The weights for MobileSAM, GroundingDINO, and PointNav must be saved to the `data/` directory. The weights can be downloaded from the following links:

- `mobile_sam.pt`:  https://github.com/ChaoningZhang/MobileSAM
- `groundingdino_swint_ogc.pth`: https://github.com/IDEA-Research/GroundingDINO
- `yolov7-e6e.pt`: https://github.com/WongKinYiu/yolov7
- `pointnav_weights.pth`: included inside the [data](data) subdirectory

## ▶️ Evaluation within Habitat

To run evaluation, various models must be loaded in the background first. This only needs to be done once by running the following command:

```bash
./scripts/launch_vlm_servers.sh
```

(You may need to run `chmod +x` on this file first.)
This command will create a tmux session that will start loading the various models used for VLFM and serving them through `flask`. When you are done, be sure to kill the tmux session to free up your GPU.

Run the following to evaluate on the HM3D dataset:

```bash
python -m vlfm.run
```

To evaluate on MP3D, run the following:

```bash
python -m vlfm.run habitat.dataset.data_path=data/datasets/objectnav/mp3d/val/val.json.gz
```

## 📰 License

VLFM is released under the [MIT License](LICENSE). This code was produced as part of Naoki Yokoyama's internship at the Boston Dynamics AI Institute in Summer 2023 and is provided "as is" without active maintenance. For questions, please contact [Naoki Yokoyama](http://naoki.io) or [Jiuguang Wang](https://www.robo.guru).

## ✒️ Citation

If you use VLFM in your research, please use the following BibTeX entry.

```
@inproceedings{yokoyama2024vlfm,
  title={VLFM: Vision-Language Frontier Maps for Zero-Shot Semantic Navigation},
  author={Naoki Yokoyama and Sehoon Ha and Dhruv Batra and Jiuguang Wang and Bernadette Bucher},
  booktitle={International Conference on Robotics and Automation (ICRA)},
  year={2024}
}
```
