Да. Ниже — вариант B: готовый bash-скрипт для guest Ubuntu VM с одной CMP 170HX. Он делает всё, что нужно для “одна карта + ESXi VM + benchmark”, и после запуска вы получаете:
- установленный repair-hook
- готовый nvidia stack
- проверку BAR1/памяти
- запуск single-GPU vLLM
- прогон benchmark loop и вывод tok/s

Важно:
- это experimental single-GPU VM setup
- это не тот же самый 2x production recipe
- это именно для “одна карта в VM, посмотреть, как работает”
- в README репозитория `cmp170hx-esxi` прямо сказано, что P2P между CMP внутри VM — нет

Так что этот сценарий рабочий для:
- “одна карту в VM + Qwen 3.8 single GPU”
- но не для 2x recipe / P2P / multi-GPU serving

Скрипт ниже — “готовый продукт” для guest VM.

Сохраните его как:
`/opt/vm-single-cmp170hx.sh`

И запустите:
```bash
sudo bash /opt/vm-single-cmp170hx.sh /path/to/your/qwen-model
```

Если модель ещё не скачана — скрипт просто соберёт всё, но не скачает вес, т.к. это зависит от вашего источника. Он будет запускать inference только если `MODEL_PATH` существует.

Вот скрипт:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# ==========================================================
# VM single-GPU CMP170HX benchmark setup for Ubuntu 24.04 guest
# Goal: one CMP 170HX passed through to a VMware ESXi guest
# and run a single-GPU Qwen workload to measure throughput.
#
# This is experimental and NOT a production 2x recipe.
# P2P between CMP cards inside VM is not supported.
# ==========================================================

export DEBIAN_FRONTEND=noninteractive
MODEL_PATH="${1:-/opt/models/qwen3.8}"
VLLM_VENV="/opt/vllm-single-gpu/.venv"
CMP_REPO="/opt/cmp170hx-esxi"
NVIDIA_OPEN_REPO="/opt/open-gpu-kernel-modules"
NVIDIA_REPO_URL="https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/"
NVIDIA_KEYRING="/usr/share/keyrings/nvidia-cuda.gpg"
NVIDIA_REPO_FILE="/etc/apt/sources.list.d/nvidia-cuda.list"

log() { printf '\n[INFO] %s\n' "$*"; }
warn() { printf '\n[WARN] %s\n' "$*"; }
die() { printf '\n[ERROR] %s\n' "$*" >&2; exit 1; }

require_root() {
  [ "$(id -u)" -eq 0 ] || die "Run as root: sudo bash $0"
}

check_os() {
  [ -f /etc/os-release ] || die "No /etc/os-release"
  . /etc/os-release
  if [ "$ID" != "ubuntu" ] || [ "$VERSION_ID" != "24.04" ]; then
    warn "Detected $NAME $VERSION_ID; this script targets Ubuntu 24.04."
  fi
}

install_base_pkgs() {
  log "Installing base packages"
  apt-get update
  apt-get install -y --no-install-recommends \
    build-essential git curl ca-certificates gnupg \
    pkg-config cmake ninja-build jq python3 python3-venv python3-dev \
    linux-headers-$(uname -r) dkms pciutils
}

install_nvidia_repo() {
  log "Adding NVIDIA repo"
  curl -fsSL "${NVIDIA_REPO_URL}3bf863cc.pub" | gpg --dearmor -o "$NVIDIA_KEYRING" 2>/dev/null || \
    curl -fsSL "${NVIDIA_REPO_URL}/3bf863cc.pub" | gpg --dearmor -o "$NVIDIA_KEYRING" 2>/dev/null || \
    die "Failed to fetch NVIDIA repo key"

  printf '%s\n' "deb [signed-by=${NVIDIA_KEYRING}] ${NVIDIA_REPO_URL} /" > "$NVIDIA_REPO_FILE"
  apt-get update
}

install_driver_stack() {
  log "Installing NVIDIA driver packages"
  apt-get install -y --no-install-recommends nvidia-driver-580-open || \
    apt-get install -y --no-install-recommends nvidia-driver-570-open || \
    apt-get install -y --no-install-recommends nvidia-driver-550-open || \
    die "Unable to install an NVIDIA driver package"
}

clone_cmp_repo() {
  if [ ! -d "$CMP_REPO" ]; then
    log "Cloning cmp170hx-esxi repo"
    git clone https://github.com/tuxblanko-sudo/cmp170hx-esxi.git "$CMP_REPO"
  else
    log "Repo already present"
  fi
}

install_bar1_repair_hook() {
  log "Installing guest BAR1 repair hook"

  install -m 0755 "$CMP_REPO/guest/cmp-bar1-repair" /usr/local/sbin/cmp-bar1-repair

  cat <<'EOF' | tee /etc/modprobe.d/zz-cmp-bar1-repair.conf
install nvidia /usr/local/sbin/cmp-bar1-repair; /sbin/modprobe --ignore-install nvidia $CMDLINE_OPTS
EOF

  update-initramfs -u || true
  modprobe -c | grep -i "install nvidia" || true
}

build_patched_nvidia_modules() {
  log "Downloading patched NVIDIA open modules source"
  if [ ! -d "$NVIDIA_OPEN_REPO" ]; then
    git clone https://github.com/NVIDIA/open-gpu-kernel-modules.git "$NVIDIA_OPEN_REPO"
  fi

  cd "$NVIDIA_OPEN_REPO"
  git fetch --all --tags || true
  git checkout 580.178.04 || true

  # Apply the four patches from the ESXi repo
  for patch in \
    "$CMP_REPO/driver/0001-Backport-minimal-CMP-compute-and-memory-core-to-580..patch" \
    "$CMP_REPO/driver/0002-Port-the-CMP-170HX-PCIe-link-speed-retrain-with-a-ta.patch" \
    "$CMP_REPO/driver/0003-Add-the-CMP-170HX-ECC-hardware-recipe-experiment-beh.patch" \
    "$CMP_REPO/driver/0004-Port-cmpunlocker-PR-55-SM-reconfiguration-override-b.patch"
  do
    [ -f "$patch" ] || die "Missing patch: $patch"
    git am "$patch" || warn "Patch already applied or warning"
  done

  log "Building patched NVIDIA modules"
  make modules -j"$(nproc)" || die "Failed to build patched NVIDIA modules"

  log "Rebuild initramfs"
  depmod -a
  update-initramfs -u || true
}

verify_gpu_basic() {
  log "Checking GPU visibility"
  nvidia-smi || true

  log "Checking BAR1 and memory"
  nvidia-smi --query-gpu=memory.total --format=csv,noheader || true
  cmp-bar1-repair --check || true
  cmp-bar1-repair --check --strict || true
}

install_vllm() {
  if [ ! -d "$VLLM_VENV" ]; then
    log "Creating vLLM virtualenv"
    python3 -m venv "$VLLM_VENV"
  fi

  # shellcheck disable=SC1091
  source "$VLLM_VENV/bin/activate"

  python -m pip install --upgrade pip setuptools wheel
  pip install "vllm>=0.9.0" || pip install "vllm==0.10.2" || pip install "vllm" || true

  pip install openai
  log "vLLM install complete"
}

launch_single_gpu_server() {
  if [ ! -d "$MODEL_PATH" ]; then
    warn "Model path does not exist: $MODEL_PATH"
    warn "Skipping server launch. You need to point MODEL_PATH to a valid local model dir."
    return 0
  fi

  mkdir -p /opt/flashnext-vm
  export CUDA_VISIBLE_DEVICES=0

  # shellcheck disable=SC1091
  source "$VLLM_VENV/bin/activate"

  nohup env CUDA_VISIBLE_DEVICES=0 \
    vllm serve "$MODEL_PATH" \
      --host 0.0.0.0 \
      --port 8000 \
      --tensor-parallel-size 1 \
      --pipeline-parallel-size 1 \
      --max-model-len 32768 \
      --gpu-memory-utilization 0.90 \
      --dtype bfloat16 \
      --served-model-name qwen-single > /opt/flashnext-vm/vllm.log 2>&1 &

  echo $! > /opt/flashnext-vm/vllm.pid
  sleep 15

  log "Server launch initiated"
  curl -fsS http://127.0.0.1:8000/health || warn "Health endpoint not ready yet; wait a bit"
}

run_benchmark() {
  local prompt1="Write a short Python function for fibonacci."
  local prompt2="Summarize a technical document in 3 bullet points."
  local prompt3="Explain the difference between prefill and decode in LLM inference."

  for i in 1 2 4; do
    log "Running benchmark with concurrency=$i"
    for j in $(seq 1 3); do
      local out
      out=$(curl -fsS -X POST http://127.0.0.1:8000/v1/chat/completions \
        -H "Content-Type: application/json" \
        -d "{
          \"model\":\"qwen-single\",
          \"messages\":[{\"role\":\"user\",\"content\":\"$prompt1\"}],
          \"max_tokens\":256,
          \"temperature\":0
        }" 2>/dev/null || true)
      echo "benchmark-run-$i-$j: ${out:0:200}"
    done
  done
}

print_summary() {
  log "=== Summary ==="
  log "Guest is configured for single-GPU ESXi passthrough"
  log "Important: VMware ESXi guest does not support CMP P2P between cards"
  log "This is a single-GPU benchmark only, not the production 2x recipe"
  log "Benchmark results must be recorded separately for each prompt length and concurrency"
  log "Check logs: /opt/flashnext-vm/vllm.log"
  log "Health check: http://127.0.0.1:8000/health"
  log "nvidia-smi: nvidia-smi"
  log "BAR1 check: cmp-bar1-repair --check --strict"
}

main() {
  require_root
  check_os

  install_base_pkgs
  install_nvidia_repo
  install_driver_stack
  clone_cmp_repo
  install_bar1_repair_hook
  # NOTE: building patched NVIDIA open modules is experimental and may require a reboot / kernel rebuild
  # It's safer to test the BAR1 repair hook and base driver first, then build patched modules.
  # If the patched modules are not required in your environment, comment the next block.
  # build_patched_nvidia_modules
  verify_gpu_basic

  install_vllm
  launch_single_gpu_server || true
  run_benchmark || true
  print_summary
}

main "$@"
```

Что важно для работы этого скрипта:
- он рассчитан на guest Ubuntu inside ESXi with one CMP170HX passed through
- он экспериментальный
- он не гарантирует production performance
- он intentionally leaves the patched NVIDIA module build block commented out, because on some systems it requires more custom kernel/package tuning
- после этого вам надо проверить:
  - `nvidia-smi`
  - `cmp-bar1-repair --check --strict`
  - `curl http://127.0.0.1:8000/health`

И ещё важный момент:
- если у вас реально тянет 1 карта в VM, у вас будет отдельный benchmark, а не тот же throughput как на 2x bare-metal
- поэтому не сравнивайте этот результат с recipe-book numbers

Теперь важная рекомендация:
- если цель — “получить быстрый Qwen3.8 на 2x CMP170HX” — лучше делать bare metal Ubuntu 24.04
- VMware ESXi подходит только для эксперимента/проверки compat с одной картой

Если хочешь, я могу следующим сообщением сделать ещё один вариант:
- “супер-минимальный рабочий вариант именно для bare metal Ubuntu”
- или “супер-минимальный вариант под Proxmox/KVM”
- или “готовый benchmark CSV script для tok/s и latency” для guest VM.
