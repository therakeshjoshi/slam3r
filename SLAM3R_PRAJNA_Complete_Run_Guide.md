# SLAM3R on PRAJNA HPC --- Complete Run Guide

Reusable guide for verifying A40 access, running SLAM3R with
already-extracted frames and local checkpoints, monitoring jobs, and
viewing the reconstruction.

## Paths and environment

-   **Repository:** `/home/watchhouse/rakeshjoshi/SLAM3R`
-   **Conda environment:** `/home/watchhouse/rakeshjoshi/envs/slam3r`
-   **Input frames:** `input_frames/` (603 JPEG frames already
    extracted)
-   **I2P checkpoint:** `checkpoints/slam3r_i2p/slam3r_i2p.pth`
-   **L2W checkpoint:** `checkpoints/slam3r_l2w/slam3r_l2w.pth`

## 1. Enter the project and activate the environment

``` bash
cd /home/watchhouse/rakeshjoshi/SLAM3R
conda activate /home/watchhouse/rakeshjoshi/envs/slam3r
```

## 2. Optional: verify A40 access separately

This small batch job requests one A40 GPU and runs `nvidia-smi`. It does
**not** run SLAM3R. The test previously succeeded as job `349022`.

``` bash
cat > ~/a40_test.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=a40_test
#SBATCH --partition=a40
#SBATCH --qos=a40
#SBATCH --gres=gpu:1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --time=00:05:00
#SBATCH --output=a40_test_%j.log

nvidia-smi
EOF

sbatch ~/a40_test.sh
squeue -u "$USER"
```

After the test job finishes, replace `JOBID` with the ID returned by
`sbatch`:

``` bash
cat ~/a40_test_JOBID.log
```

## 3. Verify existing frames and checkpoints

No frame extraction is needed when `input_frames/` already contains the
extracted images.

``` bash
cd /home/watchhouse/rakeshjoshi/SLAM3R

ls input_frames | head
find input_frames -maxdepth 1 -type f \
  \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' \) | wc -l

ls -lh checkpoints/slam3r_i2p/slam3r_i2p.pth \
       checkpoints/slam3r_l2w/slam3r_l2w.pth
```

## 4. Check the PyTorch checkpoint-loading fix

The local checkpoints are loaded by `load_model()` in `recon.py`. For
trusted checkpoint files, the relevant line should be:

``` python
ckpt = torch.load(weights, map_location=device, weights_only=False)
```

Check it with:

``` bash
grep -n 'torch.load' recon.py
```

**Security note:** `weights_only=False` permits Python pickle
deserialization. Use it only for checkpoint files from a trusted source.

## 5. Create the SLAM3R A40 batch script

This script requests one A40 GPU through SLURM, uses the existing
extracted frames and local pretrained weights, and saves prediction
outputs. It does not extract frames or download model weights.

``` bash
cd /home/watchhouse/rakeshjoshi/SLAM3R

cat > run_slam3r_a40.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=slam3r_video
#SBATCH --partition=a40
#SBATCH --qos=a40
#SBATCH --gres=gpu:1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=slam3r_%j.log
#SBATCH --error=slam3r_%j.err

set -e

cd /home/watchhouse/rakeshjoshi/SLAM3R
source ~/.bashrc
conda activate /home/watchhouse/rakeshjoshi/envs/slam3r

echo "===== GPU INFORMATION ====="
nvidia-smi

echo "===== CHECKING INPUT FRAMES ====="
test -d input_frames
test "$(find input_frames -maxdepth 1 -type f \
  \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' \) | wc -l)" -gt 0

echo "===== CHECKING LOCAL CHECKPOINTS ====="
test -f checkpoints/slam3r_i2p/slam3r_i2p.pth
test -f checkpoints/slam3r_l2w/slam3r_l2w.pth

echo "===== RUNNING SLAM3R ====="
python recon.py \
  --test_name PRAJNA_input_video \
  --dataset "$PWD/input_frames" \
  --gpu_id 0 \
  --i2p_weights "$PWD/checkpoints/slam3r_i2p/slam3r_i2p.pth" \
  --l2w_weights "$PWD/checkpoints/slam3r_l2w/slam3r_l2w.pth" \
  --keyframe_stride 3 \
  --win_r 5 \
  --num_scene_frame 10 \
  --initial_winsize 5 \
  --conf_thres_l2w 12 \
  --conf_thres_i2p 1.5 \
  --num_points_save 1000000 \
  --update_buffer_intv 1 \
  --buffer_size 100 \
  --buffer_strategy reservoir \
  --max_num_register 10 \
  --keyframe_adapt_min 1 \
  --keyframe_adapt_max 20 \
  --keyframe_adapt_stride 1 \
  --retrieve_freq 1 \
  --save_preds

echo "===== SLAM3R FINISHED SUCCESSFULLY ====="
EOF
```

### What the SLURM resource settings mean

  -----------------------------------------------------------------------
  Setting                             Meaning
  ----------------------------------- -----------------------------------
  `--partition=a40`                   Schedule on A40 nodes

  `--qos=a40`                         Use the A40 QoS

  `--gres=gpu:1`                      Request one GPU

  `--cpus-per-task=4`                 Request four CPU cores

  `--mem=32G`                         Request 32 GB system RAM

  `--time=04:00:00`                   Set a four-hour time limit

  `--output` / `--error`              Save standard output and errors to
                                      separate log files
  -----------------------------------------------------------------------

Interactive jobs are not permitted on the A40 partition. The `#SBATCH`
directives are the resource request; there is no need to request an
interactive GPU shell first.

## 6. Submit the SLAM3R job

``` bash
sbatch run_slam3r_a40.sh
```

The command prints a job ID, for example `349060`. SLURM queues the job
and assigns an eligible A40 node when resources and account limits allow
it.

## 7. Monitor the job

Replace `JOBID` with the actual ID returned by `sbatch`.

``` bash
squeue -u "$USER"
squeue -j JOBID
tail -f slam3r_JOBID.log
cat slam3r_JOBID.err
sacct -j JOBID --format=JobID,JobName,Partition,State,Elapsed,ExitCode
```

A successful completed job should report `COMPLETED` and exit code
`0:0`. If the job has left the queue, use `sacct` and inspect its logs.

## 8. Example of successful output

The successful run reported:

``` text
No running processes found
===== CHECKING INPUT FRAMES =====
===== CHECKING LOCAL CHECKPOINTS =====
===== RUNNING SLAM3R =====
Warning, cannot find cuda-compiled version of RoPE2D, using a slow pytorch version instead
xformers not avaliable, use self-implemented attention instead
using gpu:  0
Loading pretrained:  /home/watchhouse/rakeshjoshi/SLAM3R/checkpoints/slam3r_i2p/slam3r_i2p.pth
<All keys matched successfully>
Loading pretrained:  /home/watchhouse/rakeshjoshi/SLAM3R/checkpoints/slam3r_l2w/slam3r_l2w.pth
<All keys matched successfully>
Loading dataset:  /home/watchhouse/rakeshjoshi/SLAM3R/input_frames
(603 images loaded)
finish pre-extracting img tokens
initialize scene with 5 views, with a mean confidence of 13.69
finish recovering pcds of 603 frames in their local coordinates, with a mean confidence of 5.34
finish aligning 12 head frames, with a max mean confidence of 77.88
mean confidence for whole scene reconstruction: 34.49
ratio of points filtered out: 25.08%
resampling 1000000 points from 22668738 points
>> save_to results/PRAJNA_input_video/SLAM3R_input_frames_recon.ply
>> saving per-frame predictions to results/PRAJNA_input_video/preds
===== SLAM3R FINISHED =====
```

The RoPE2D and xFormers messages are performance warnings in this run.
The log confirms that both checkpoints loaded and the point cloud and
per-frame predictions were saved.

## 9. Output files

Main reconstructed point cloud:

``` text
results/PRAJNA_input_video/SLAM3R_input_frames_recon.ply
```

Per-frame predictions:

``` text
results/PRAJNA_input_video/preds/
```

Check the output files:

``` bash
ls -lh results/PRAJNA_input_video/
ls -lh results/PRAJNA_input_video/preds/ | head
```

## 10. View the point cloud in a desktop window

The `.ply` file is a 3D point cloud, **not a video**. To inspect it on
your Windows computer, copy it from PRAJNA using PowerShell. Replace
`PRAJNA_HOST` with the SSH hostname you normally use to connect to the
cluster:

``` powershell
scp rakeshjoshi@PRAJNA_HOST:/home/watchhouse/rakeshjoshi/SLAM3R/results/PRAJNA_input_video/SLAM3R_input_frames_recon.ply .
```

Open the downloaded `.ply` in
[CloudCompare](https://www.cloudcompare.org/) or MeshLab. In
CloudCompare, use **File → Open**, select the `.ply`, and rotate/zoom to
inspect the reconstruction. To create a rotating video, record the
viewer or render a separate visualization.

## 11. Run another experiment

Change `--test_name` in the batch script to a unique value before
submitting again. For example:

``` bash
--test_name PRAJNA_input_video_run02
```

Then submit:

``` bash
sbatch run_slam3r_a40.sh
```

Use a distinct `test_name` for each run to avoid output collisions.
Reuse `input_frames/` and the local checkpoints; do not extract frames
again unless the input video changes.
