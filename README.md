# Wildfire-Prediction-ChaosBench

Running ChaosBench training on Colab (T4) and on the LEAP JupyterHub: setup, fixes, and timings.

## Colab (free tier)
- Tesla T4, 15 GB GPU memory, 2 CPU cores, about 112 GB disk, Python 3.13
- Notebook: ChaosBench.ipynb

## LEAP JupyterHub
- Server: GPU option with the Pangeo PyTorch ML Notebook image
- One Tesla T4 (16 GB VRAM), 8 CPUs
- Cold start took about 10 minutes, and the first attempt timed out at 600 seconds while the image was pulling. Starting again was faster.
- GPU is the only GPU type offered

## Problems and fixes
- The pinned requirements.txt fails on Python 3.13 (numpy 1.25.2 cannot be built). Fix: install with the version pins removed:
  sed -E 's/==.*//' requirements.txt > req_unpinned.txt
  pip install -r req_unpinned.txt
  pip install zarr
- On the hub the PyTorch image already has most packages, so the install takes seconds.
- process.sh cannot download a subset and needs about double the compressed size in free disk while it joins chunks.
- curl is not installed on the hub image. Fix: download with Python and extract while streaming.
- The data download was cut off once when streamed directly into tar. Fix: save the chunks to disk first, then extract.

## Data
- ERA5: 41 GB compressed, about 51 GB extracted, 16,436 daily zarr files (1979 to 2023), 673,876 files in total
- Climatology: 9.2 GB compressed, about 11 GB extracted, 8,077 files
- Both are in the cloud bucket at gs://leap-persistent/maryamfarhan/chaosbench/
- Upload with gcloud storage cp -r took about 21 minutes for ERA5 (38 MiB/s) and a few minutes for climatology (103 MiB/s)
- /tmp is wiped when the server stops, so restore ERA5 from the bucket after a restart

## Test training run (hub)
Config: unet_s2s with 1 epoch, batch size 4, train years 1979 to 1981, validation year 2016, 8 workers.

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

Run: python train.py --config_filepath chaosbench/configs/unet_test_s2s.yaml

Result: 274 batches in 52 seconds, train loss 0.415, validation loss 0.364. Colab gave the same losses at about 3 minutes per epoch.

## Rough estimate for the full config
The full unet_s2s config uses 37 training years, so about 10 minutes per epoch on one T4 by extrapolation, or around 90 hours for 500 epochs. This is a guess and not a measurement.
