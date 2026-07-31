# Nemotron 3 Super NIM benchmarking with AIPerf

This customer-shareable example runs repeatable input sequence length (ISL) × output sequence length (OSL) tests against an already-running, OpenAI-compatible NVIDIA NIM. It is specific to `nvidia/nemotron-3-super-120b-a12b` and uses the tokenizer from the local NIM cache.

AIPerf captures client latency and throughput plus benchmark-window NIM metrics. Prometheus independently scrapes NIM every two seconds for the provisioned near-real-time Grafana dashboard.

## Package contents

- `notebooks/nim_aiperf_benchmark.ipynb`: validation, observability startup, smoke test, native AIPerf grid sweep, annotations, and result indexing.
- `.env.example`: customer-editable endpoint, tokenizer, Kubernetes, observability, and offered-load settings.
- `compose.observability.yaml`: local Prometheus and Grafana services.
- `compose.benchmark.yaml` and `Dockerfile`: optional containerized Jupyter/AIPerf deployment layered on the same observability services.
- `observability/`: Prometheus configuration and provisioned Grafana dashboard.
- `k8s/profiles/`: an optional tuned comparison profile; it is not used for the default-runtime baseline.

The NIM container, model weights, tokenizer, credentials, and benchmark results are not included.

## 1. Configure the environment

Create a private local configuration:

```bash
cp .env.example .env
```

Edit at least these values:

```dotenv
NIM_TOKENIZER_PATH=/absolute/path/to/the/tokenizer/in/your/local/NIM/cache
K8S_NAMESPACE=your-namespace
K8S_NIM_SERVICE=your-nim-service
K8S_NIM_DEPLOYMENT=your-nim-deployment
```

`.env` and benchmark results are ignored. Do not put credentials in `.env.example` or commit a populated `.env`.

## 2. Install the notebook environment

```bash
conda create -n bench python=3.12 -y
conda activate bench
python -m pip install -r requirements.txt
python -m ipykernel install --user --name bench --display-name "Python (bench)"
```

## 3. Expose the running NIM locally

From the repository root:

```bash
set -a
source .env
set +a
kubectl -n "$K8S_NAMESPACE" port-forward --address "$NIM_BIND_ADDRESS" service/"$K8S_NIM_SERVICE" "${NIM_LOCAL_PORT}:8000"
```

The default `.env.example` routes both AIPerf and the Prometheus container through this tunnel. Because Docker reaches the host through `host.docker.internal`, that example binds the tunnel to `0.0.0.0`; use a host firewall and do not expose an unauthenticated NIM port to an untrusted network. If Docker can directly reach a Kubernetes service address, set `NIM_PROMETHEUS_TARGET` to that address and bind the tunnel to `127.0.0.1` instead. For a remote endpoint, set `NIM_BASE_URL`, `NIM_METRICS_URL`, and a `NIM_PROMETHEUS_TARGET` address reachable from inside Docker.

## 4. Test NIM runtime defaults

The baseline intentionally does not set any of the following:

- `NIM_MAX_MODEL_LEN` or vLLM `--max-model-len`
- `NIM_MAX_BATCH_SIZE`
- vLLM `--max-num-seqs`

Confirm those overrides are absent from the NIM deployment before testing. The notebook reads `/v1/models` to report the context length actually resolved by NIM. AIPerf’s `AIPERF_CONCURRENCY`, ISL, and OSL settings only define offered client load; they do not change the server context window or scheduler limits.

The included `k8s/profiles/nemotron-3-super-throughput-8k-16seq.patch.yaml` explicitly sets 8K/16 and is retained only for a later tuned comparison. Do not apply it for the default-runtime baseline.

## 5. Run the notebook

```bash
conda activate bench
jupyter lab notebooks/nim_aiperf_benchmark.ipynb
```

Select `Python (bench)`, run the configuration and preflight cells, start observability, then run the smoke test. Keep `RUN_FULL_SWEEP=false` until the endpoint and Grafana telemetry are verified.

Workload controls live in `.env`:

- `AIPERF_ISL_VALUES` and `AIPERF_OSL_VALUES` define the sweep shapes.
- `AIPERF_SWEEP_TYPE=grid` runs their Cartesian product; `zip` pairs values.
- `AIPERF_CONCURRENCY` is the number of simultaneous client requests offered to NIM.
- `AIPERF_FORCE_EXACT_OSL=true` sends `ignore_eos:true` so compatible vLLM backends reach the requested OSL.

Ensure each requested `ISL + OSL` is within the context length reported by the running NIM.

## Observability only

```bash
docker compose --env-file .env -f compose.observability.yaml up -d
```

Open the `GRAFANA_URL` configured in `.env`; the dashboard path is `/d/nim-llm-overview`. Stop while preserving retained data:

```bash
docker compose --env-file .env -f compose.observability.yaml down
```

Use `down -v` only when intentionally deleting Prometheus and Grafana volumes.

## Optional Docker/Compose benchmark workspace

This alternative runs Jupyter, AIPerf, Prometheus, and Grafana in Compose, so a local Conda environment is not required. It still targets an already-running NIM; it does not deploy the model server or include model weights.

Set `NIM_TOKENIZER_PATH` in `.env` to the absolute host path of the tokenizer. If NIM is not reachable from containers at `host.docker.internal:18000`, also set `DOCKER_NIM_BASE_URL`, `DOCKER_NIM_METRICS_URL`, and `NIM_PROMETHEUS_TARGET` to container-reachable addresses. Set `BENCHMARK_UID`/`BENCHMARK_GID` to the host user numeric IDs when they are not `1000`.

Build and start the full workspace:

```bash
docker compose --env-file .env \
  -f compose.observability.yaml -f compose.benchmark.yaml \
  up -d --build
```

Open `http://localhost:8888/lab?token=aiperf` (or use the `JUPYTER_PORT` and `JUPYTER_TOKEN` configured in `.env`), select `notebooks/nim_aiperf_benchmark.ipynb`, and run it as described above. The repository is bind-mounted at `/workspace`, so results persist in the host `benchmark_results/` directory.

Check service health and logs:

```bash
docker compose --env-file .env \
  -f compose.observability.yaml -f compose.benchmark.yaml ps
docker compose --env-file .env \
  -f compose.observability.yaml -f compose.benchmark.yaml logs benchmark
```

Stop the workspace while preserving Grafana and Prometheus data:

```bash
docker compose --env-file .env \
  -f compose.observability.yaml -f compose.benchmark.yaml down
```

## Measurement notes

- Streaming is enabled so AIPerf can measure TTFT and inter-token latency.
- Synthetic prompts characterize serving behavior but do not replace a representative production dataset.
- AIPerf server-metric exports are benchmark-window artifacts; Prometheus/Grafana provide live viewing and retained history.
- A high model context limit is a per-request admission limit, not a reservation of that many KV-cache tokens for every request.

## References

- [NVIDIA NIM AIPerf benchmarking guide](https://docs.nvidia.com/nim/benchmarking/llm/latest/quickstart.html)
- [NVIDIA NIM logging and observability](https://docs.nvidia.com/nim/large-language-models/latest/reference/logging-and-observability.html)
- [AIPerf](https://github.com/ai-dynamo/aiperf)
- [AIPerf parameter sweeps](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/sweeps.md)
- [AIPerf server metrics](https://github.com/ai-dynamo/aiperf/blob/main/docs/server-metrics/server-metrics.md)
