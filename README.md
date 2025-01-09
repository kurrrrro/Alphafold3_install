# AlphaFold3 (Docker) のインストール備忘録

(2024.01.08)

## はじめに

今後新たに構築する場合は必ず公式HPの手順に則ること。またAlphafold3 installationの公式HPのまま実行してもエラーが出るときは、Alphafold3公式HPで参照されている引用元を見るべし。
この備忘録は下記公式HPの方法を、ラボ内のパソコンで動作するように改変したのものである。
超初心者によるメモ書きのため、動作等すべての責任は一切負わない。適宜改変してほしい。

公式HP [Alphafold3のインストール方法](https://github.com/google-deepmind/alphafold3/blob/main/docs/installation.md)

## システム要件

- Ubuntu-22.04 LTS
- RAM 64 GB
- SSD =< 650GB
- CUDA 12.6.0の動作するNVIDIA製GPU
- `curl` `git` `wget` `ztsd`コマンドが既にインストール済み
  未インストールの場合は下記を実行

  ```bash
  sudo apt -y install curl git wget ztsd
  ```
- AlphaFold3 model parametersのファイル
  ※事前に申請が必要。
  　[Google Form](https://docs.google.com/forms/d/e/1FAIpQLSfWZAgo1aYk0O4MuAXZj8xRQ8DafeFJnldNOnh_13qAx2ceZw/viewform?pli=1)から申請すると、2~3営業日でダウンロードリンクがメールに送付される。

## 公式インストール方法 (Docker用)

> 公式HP [Alphafold3のインストール方法](https://github.com/google-deepmind/alphafold3/blob/main/docs/installation.md)

#### 1. gccコンパイラの確認

　Ubuntu22.04でGCC11コンパイラをインストール

```bash
apt -y install gcc g++
```

　インストールが完了しているか確認

```bash
gcc --version
```

#### 2. NVIDIAドライバーのインストール

> 公式HP [NVIDIA deivers installation](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/)

　サポートバージョンの確認

```bash
uname -m && cat /etc/<os-release>
```

　NVIDIAドライバーのインストール

> 公式HP [Datacenter Driver 535 Downloads](https://developer.nvidia.com/datacenter-driver-downloads)

　自身のPC環境に合ったものを選ぶこと。下記は一例である。

```bash
wget https://developer.download.nvidia.com/compute/nvidia-driver/535.216.03/local_installers/nvidia-driver-local-repo-ubuntu2204-535.216.03_1.0-1_amd64.deb
sudo dpkg -i nvidia-driver-local-repo-ubuntu2204-535.216.03_1.0-1_amd64.deb
sudo cp /var/nvidia-driver-local-repo-ubuntu2204-535.216.03/nvidia-driver-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
```

```bash
sudo apt-get install -y nvidia-kernel-open-535
sudo apt-get install -y cuda-drivers-535
```

　CUDA 12.6.0 のインストール

> 公式HP [CUDA Toolkit 12.6 Downloads](https://developer.nvidia.com/cuda-12-6-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=deb_local)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.6.0/local_installers/cuda-repo-ubuntu2204-12-6-local_12.6.0-560.28.03-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2204-12-6-local_12.6.0-560.28.03-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2204-12-6-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-6
```

```bash
sudo apt-get install -y nvidia-open
```

　Cuda compilerの確認

```bash
nvcc --version
```

　CUDA のバージョンが12.6になっていることを確認して次へすすむ

#### 3. Dockerのインストール

　Ubuntu 22.04 LTS イメージにのみ適用

> 公式HP [Docker Installation methods](Installation methods)

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

　エラーが出なければ、ルートレスDockerの有効化

```bash
sudo apt-get install -y uidmap systemd-container

sudo machinectl shell $(whoami)@ /bin/bash -c 'dockerd-rootless-setuptool.sh install && sudo loginctl enable-linger $(whoami) && DOCKER_HOST=unix:///run/user/1001/docker.sock docker context use rootless'
```

#### 4. Docker用 NVIDIAサポートのインストール

> 公式HP [Installing the NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

　Aptでインストールする

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```bash
sed -i -e '/experimental/ s/^#//g' /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

```bash
sudo apt-get update
```

```bash
sudo apt-get install -y nvidia-container-toolkit
```

　ルートレスモードの設定

```bash
nvidia-ctk runtime configure --runtime=docker --config=$HOME/.config/docker/daemon.json
```

　Rootless Dockerデーモンの再起動

```bash
systemctl --user restart docker
```

```bash
sudo nvidia-ctk config --set nvidia-container-cli.no-cgroups --in-place
```

　Dockerコンテナでのリソース管理におけるCgroupsの有効化

```bash
sudo nano /etc/nvidia-container-runtime/config.toml
```

　開いた画面で `no-cgroups`を下記のように変更

```
no-cgroups = false
```

　Ctrl + X, Yes, Enterを押下し、編集を完了させる。

　エラーが出なければ、Dockerでサンプルワークロードを実行

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

　GPUが認識されていればOK

#### 5. AlphaFold3ソースコードの入手

```bash
git clone https://github.com/google-deepmind/alphafold3.git
```

#### 6. 遺伝子データベースの入手

　２種類のパターンを示した。なお、ダウンロードしたデータはAlphaFold3のサブディレクトリに入れず、home直下に配置する。

　※AlphaFold3のサブディレクトリ内に配置した場合、解析に非常に時間がかかるようになるので注意。

***＜外部SSDに保存する場合＞***

　任意の外部記憶媒体に、databaseという名前のフォルダを作成する。

　databaseフォルダの中に新しくpublic_databasesという名のフォルダを作成する。

```bash
.
└── database/
    └── public_databases
```

　ディレクトリデータベースの指定

```bash
DB_DIR="/mnt/database/public_databases"
```

　データベースのダウンロード

```bash
cd alphafold3
chmod +x fetch_databases.sh
./fetch_databases.sh $DB_DIR
```

　データが激重なので、下手したらダウンロードに一晩かかります。

　***＜ローカルSSDに保存する場合＞***

ディレクトリデータベースの指定

```bash
DB_DIR="home/user/public_databases"
```

　データベースのダウンロード

```bash
cd alphafold3
chmod +x fetch_databases.sh
./fetch_databases.sh $DB_DIR
```

home/user/直下にpublic_databasesという名前のディレクトリを移動させる。

```bash
.
└── home/
    └── user/
        └── public/databases
```

#### 7. モデルパラメータの取得および設置

　※事前に申請が必要。
　　メールに添付されたダウンロードリンクからダウンロード・解凍する。

　/home/user/ 直下にmodels という名前のディレクトリを作成する。

 　modelsディレクトリの中に、解凍済みモデルパラメータを配置する。

```bash
.
└── home/
    └── user/
        └── models/
            └── 解凍済みパラメーターファイル
```

 　モデルパラメータの指定

```bash
MODEL_PARAMETERS_DIR="home/user/models/解凍済みパラメーターファイル名"
```

#### 8. AlphaFold3を実行するDockerコンテナの構築

```bash
sudo docker build -t alphafold3 -f docker/Dockerfile .
```

#### 9. 実行ファイルの準備

> [json ファイルの作成方法](https://github.com/google-deepmind/alphafold3?tab=readme-ov-file#installation-and-running-your-first-prediction)

　予測したい対象を入力したjsonファイルを作成。

　inputファイルのディレクトリおよびoutputファイル出力ディレクトリを作成する。

　ex ) /home/user/ 直下にaf_input, af_outputという名前のディレクトリを作成する。

```bash
.
└── home/
    └── user/
        ├── af_input
        └── af_output
```

　作成したjsonファイルを af_input ディレクトリに入れる。

#### 10. AlphaFold3の実行

```bash
sudo docker run -it \
    --volume $HOME/af_input:/root/af_input \
    --volume $HOME/af_output:/root/af_output \
    --volume MODEL_PARAMETERS_DIR:/root/models \
    --volume DB_DIR:/root/public_databases \
    --gpus all \
    alphafold3 \
    python run_alphafold.py \
    --json_path=/root/af_input/xxxxxx.json \
    --model_dir=/root/models \
    --output_dir=/root/af_output
```
