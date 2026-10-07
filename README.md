# ai-infra-evaluator

A single-script benchmark that launches EC2 instances of the types you name, sends ResNet50 inference requests to a TorchServe container on each, and plots average latency against cost per inference. It is for comparing instance types on the same model with a back-of-envelope on-demand price, not for production capacity planning.

## What it does not do

- It does not use the GPU. `user_data.sh` starts the `pytorch/torchserve:latest-cpu` image on every instance, including `g4dn.xlarge`, so a GPU instance type is measured running a CPU container.
- It does not supply a model. The container is started with `--models resnet50=mar-resnet50.mar`, and nothing in this repo creates or mounts that archive. Not documented: where the `.mar` file is meant to come from, so as written the service may never become ready.
- It does not look up prices. `PRICES` in `benchmark.py` holds two hard-coded hourly rates (`t3.medium` 0.0416, `g4dn.xlarge` 0.526 USD) and raises `KeyError` for any other instance type.
- It does not create a security group, subnet or key pair. Port 8080 on the instance must be reachable from where you run the script, which depends on your default VPC and security group.
- Cost ignores startup time, storage and data transfer. It is hourly rate times measured request time.

## Quickstart

Not verified end to end: it launches paid EC2 instances and I did not run it. What was checked on 2026-10-07: `--help`, and `cost_per_inference` and `plot_results` run on made-up numbers (for example 0.05 s average latency over 100 runs on `g4dn.xlarge` gives 7.3e-06 USD per inference).

```bash
python3 -m venv venv && source venv/bin/activate
pip install boto3 requests matplotlib
python benchmark.py --instances t3.medium g4dn.xlarge --runs 50
```

`requirements.txt` also lists `torch` and `torchserve`, which `benchmark.py` does not import; installing only the three packages above is enough. Before running:

1. Put AWS credentials in your environment with permission for `RunInstances`, `DescribeInstances` and `TerminateInstances`.
2. Replace the hard-coded AMI (`ami-0b3ceb28d1a07fa60`) in `launch_instance()` with an image that has Docker and `yum`/`amazon-linux-extras` (the user data script assumes Amazon Linux 2) and exists in your region.
3. Set your region. `launch_instance` uses your default boto3 region while the waiters use `us-east-1`; keep them the same.

The script writes `results.png` in the current directory.

## How it works

For each instance type, `benchmark.py` launches one instance with `user_data.sh` as user data, waits for the `running` state, reads its public DNS name, and polls `http://<host>:8080/ping` for up to 300 seconds. It then POSTs `sample.png` (a 32 by 32 image) to `/predictions/resnet50` the requested number of times, one request at a time, and averages the wall-clock latency. Cost per inference is the hourly rate times total request time divided by 3600 and by the run count.

Termination sits in a `finally` block, so an instance is terminated after a failure in the wait or benchmark steps too. If the script is killed before that, or `run_instances` succeeded but the process died, check the console for a leftover instance named `ai-evaluator`. The script sets only the tag `Name=ai-evaluator`.

```
benchmark.py     CLI, EC2 lifecycle, request loop, cost math, plot
user_data.sh     installs Docker, starts the TorchServe CPU container
sample.png       32x32 test image sent as the request body
```

## Known limits

- Requests are sequential from your machine, so latency includes your network path to AWS, not only inference time.
- One instance at a time, no retries, and no results saved except the plot.
- The plot is a bar chart of latency with a line for cost over a categorical axis.

## Status

Built in 2025 as a small prototype. Not verified end to end.

## License

MIT. See `LICENSE`.
