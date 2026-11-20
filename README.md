# Wildfire-Prediction-ChaosBench

Running ChaosBench training on Colab (T4) and on the LEAP JupyterHub: setup, fixes, and timings. Tested on 20 November 2026.

## Environments

### Colab (free tier)
- Tesla T4 (15 GB GPU memory), 2 CPU cores, about 112 GB disk, Python 3.13
- Notebook: ChaosBench.ipynb

### LEAP JupyterHub
- Server option: GPU, with the Pangeo PyTorch ML Notebook image (2026.06.04)
- One Tesla T4 (16 GB VRAM), 8 CPUs. The T4 is the only GPU type offered.
- Cold start took about 10 minutes. The first attempt timed out at 600 seconds while the image was pulling, and starting again was faster.
- Python 3.13, PyTorch 2.10, and most required packages are already in the image

## Problems and fixes
- The pinned requirements.txt fails on Python 3.13 because numpy 1.25.2 cannot be built. Fix: install with the version pins removed.

```
sed -E 's/==.*//' requirements.txt > req_unpinned.txt
pip install -r req_unpinned.txt
pip install zarr
```

- On the hub this install takes seconds, because the image already has most packages.
- process.sh cannot download a subset. It joins all chunks into one archive before extracting, so it needs about double the compressed size in free disk.
- curl is not installed on the hub image. Fix: download with Python and pipe the stream into tar.
- Streaming the climatology download straight into tar broke at 3.2 GB (gzip unexpected end of file). Fix: save each chunk to disk, check its size, then extract.
- A bucket listing can take several minutes with this many small files. Compare file counts instead of running du.

## Data
- ERA5: 41 GB compressed, about 51 GB extracted, 16,436 daily zarr files (1979 to 2023), 673,876 files in total
- Climatology: 9.2 GB compressed, about 11 GB extracted, 8,077 files
- Both are in the cloud bucket at gs://leap-persistent/maryamfarhan/chaosbench/ and the file counts match the local copies (ERA5: 16,436 daily folders, climatology: 8,077 files)
- Upload with gcloud storage cp -r: about 21 minutes for ERA5 (38 MiB/s) and a few minutes for climatology (103 MiB/s)
- /tmp is wiped when the server stops. ERA5 lives in /tmp because the home folder is shared and nearly full, so restore it from the bucket after a restart.

## How teammates get the data
Run these on your own LEAP hub server (gcloud is already logged in there). First check that you can read the bucket:

```
gcloud storage ls gs://leap-persistent/maryamfarhan/chaosbench/
```

You should see era5/ and climatology/. Then copy the data and link it into the repo:

```
mkdir -p /tmp/chaosdata
gcloud storage cp -r gs://leap-persistent/maryamfarhan/chaosbench/era5 /tmp/chaosdata/
gcloud storage cp -r gs://leap-persistent/maryamfarhan/chaosbench/climatology /tmp/chaosdata/
mkdir -p ~/ChaosBench/data
ln -sfn /tmp/chaosdata/era5 ~/ChaosBench/data/era5
ln -sfn /tmp/chaosdata/climatology ~/ChaosBench/data/climatology
```

/tmp is wiped when your server stops, so repeat this after a restart. If you get "permission denied" on the first command, tell Maryam.

## Run order on the hub
1. Start the GPU server (PyTorch image) and open a terminal.
2. Clone the repo and install the requirements (see Problems and fixes).
3. Get the data (see How teammates get the data, or stream the download from Hugging Face).
4. Link both folders into ChaosBench/data as era5 and climatology.
5. Write the test config below and run training in the background.

Restore ERA5 only after a restart:

```
mkdir -p /tmp/chaosdata
gcloud storage cp -r gs://leap-persistent/maryamfarhan/chaosbench/era5 /tmp/chaosdata/
ln -sfn /tmp/chaosdata/era5 ~/ChaosBench/data/era5
```

## Test training run (hub)
Config: unet_s2s, 1 epoch, batch size 4, train years 1979 to 1981, validation year 2016, 8 workers.

```
model_args:
    model_name: 'unet_s2s'
    input_size: 60
    output_size: 60
    learning_rate: 0.01
    num_workers: 8
    epochs: 1
    t_max: 1
    only_headline: False
data_args:
    batch_size: 4
    train_years: [1979, 1980, 1981]
    val_years: [2016]
    n_step: 1
    lead_time: 1
    land_vars: []
    ocean_vars: []
```

Run:

```
cd ~/ChaosBench
nohup python train.py --config_filepath chaosbench/configs/unet_test_s2s.yaml > ~/train.log 2>&1 &
```

Result: 274 batches in 52 seconds on the hub (about 3 minutes on Colab), train loss 0.415, validation loss 0.364. Both environments gave the same losses.

## Rough estimate for the full config
The full unet_s2s config uses 37 training years, which is about 10 minutes per epoch on one T4 by extrapolation, or around 90 hours for 500 epochs. This is a guess and has not been measured. Peak GPU memory has not been measured either.
