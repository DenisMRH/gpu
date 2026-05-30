# Отчет по лабораторной работе: GPU-кластер Kubernetes

Все команды из отчета выполнять на VM. Чтобы относительные пути работали корректно, перед началом перейти в каталог лабораторной:

```bash
cd /home/mrstart151/k8s-gpu-lab
```

Если команда запускается из другого каталога, использовать абсолютные пути из примеров ниже.

## 1. Параметры VM и GPU

| Параметр | Значение |
|----------|----------|
| ОС | Ubuntu 22.04.5 LTS |
| Ядро | 5.15.0-157-generic |
| CPU | 8 vCPU |
| RAM | около 32 GiB |
| GPU | NVIDIA Tesla T4, 15360 MiB VRAM |
| Kubernetes | v1.34.8 |
| containerd | v2.2.1 |
| GPU Operator | v25.10.1 |
| NVIDIA Driver | 580.105.08 |
| CUDA в `nvidia-smi` | 13.0 |
| CUDA runtime в PyTorch image | 12.4 |

## 2. Состояние Kubernetes-кластера

### Скриншот 1. Узел Kubernetes

Выполнить команду:

```bash
kubectl get nodes -o wide
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

- node `epd04l4uglackuqkb8j9`;
- статус `Ready`;
- Kubernetes `v1.34.8`;
- Ubuntu `22.04.5 LTS`;
- containerd `2.2.1`.

Ожидаемый результат:

```text
NAME                   STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
epd04l4uglackuqkb8j9   Ready    control-plane   11d   v1.34.8   10.129.0.33   <none>        Ubuntu 22.04.5 LTS   5.15.0-157-generic   containerd://2.2.1
```

### Скриншот 2. Все pod'ы в кластере

Выполнить команду:

```bash
kubectl get pods -A
```

Сделать скриншот после появления вывода. Если вывод не помещается на экран, сделать 2 скриншота: верхнюю и нижнюю часть. На скриншоте должны быть видны namespace:

- `gpu-operator`;
- `gpu-lab`;
- `monitoring`;
- `kube-system`;
- `kube-flannel`.

Ожидаемое состояние: системные pod'ы и pod'ы GPU Operator находятся в `Running`, benchmark/stress job в `Completed`.

### Скриншот 3. GPU resource на node

Выполнить команду:

```bash
kubectl get node epd04l4uglackuqkb8j9 -o jsonpath='{.status.capacity}'
```

Сделать скриншот после появления вывода. На скриншоте обязательно должно быть видно:

```text
"nvidia.com/gpu":"1"
```

Ожидаемый результат:

```json
{"cpu":"8","ephemeral-storage":"100941264Ki","hugepages-1Gi":"0","hugepages-2Mi":"0","memory":"32861524Ki","nvidia.com/gpu":"1","nvidia.com/gpu.shared":"0","pods":"110"}
```

### Скриншот 4. Helm releases

Выполнить команду:

```bash
helm list -A
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

- `gpu-operator` в namespace `gpu-operator`, chart `gpu-operator-v25.10.1`;
- `monitoring` в namespace `monitoring`, chart `kube-prometheus-stack`.

## 3. Проверка NVIDIA GPU внутри Kubernetes

Важно: в этой лабораторной драйвер устанавливает NVIDIA GPU Operator в driver container, потому что GPU Operator был установлен с `driver.enabled=true`. Поэтому команда `nvidia-smi` на host VM может не существовать, и это нормально:

```bash
nvidia-smi
sudo nvidia-smi
```

Ожидаемый host-результат в этой конфигурации:

```text
Command 'nvidia-smi' not found
sudo: nvidia-smi: command not found
```

GPU нужно проверять внутри Kubernetes driver daemonset.

### Скриншот 5. `nvidia-smi` из driver daemonset

Выполнить команду:

```bash
kubectl exec -n gpu-operator ds/nvidia-driver-daemonset -- nvidia-smi
```

Сделать скриншот после появления таблицы `nvidia-smi`. На скриншоте должно быть видно:

- `NVIDIA-SMI 580.105.08`;
- `Driver Version: 580.105.08`;
- `CUDA Version: 13.0`;
- GPU `Tesla T4`;
- memory `15360 MiB`;
- текущий idle state без running processes.

Ожидаемый смысл результата: Kubernetes driver daemonset видит физическую GPU.

## 4. Проверка CUDA через JupyterLab

В manifest `/home/mrstart151/k8s-gpu-lab/gpu-lab/jupyterlab.yaml` для JupyterLab указан `dnsPolicy: Default`, потому что в текущем окружении pod DNS через `kube-dns` может мешать установке `jupyterlab` через `pip`.

Если браузер открыт на локальном компьютере, а Kubernetes-команды выполняются на VM по SSH, удобнее открыть SSH сразу с tunnel. В локальном терминале на своем компьютере выполнить:

```bash
ssh -L 8888:127.0.0.1:8888 mrstart151@51.250.20.172
```

Если локальный порт `8888` уже занят и появляется `Address already in use`, использовать локальный порт `8889`:

```bash
ssh -L 8889:127.0.0.1:8888 mrstart151@51.250.20.172
```

Дальше в этом же SSH-терминале на VM запустить JupyterLab:

```bash
cd /home/mrstart151/k8s-gpu-lab
kubectl apply -f /home/mrstart151/k8s-gpu-lab/gpu-lab/jupyterlab.yaml
kubectl rollout status deployment/jupyterlab -n gpu-lab --timeout=10m
kubectl get pods -n gpu-lab -l app=jupyterlab -o wide
```

Когда pod станет `Running` и deployment станет `successfully rolled out`, в этом же SSH-терминале выполнить port-forward и оставить команду запущенной:

```bash
kubectl port-forward -n gpu-lab svc/jupyterlab 8888:8888
```

Открыть в локальном браузере:

```text
http://localhost:8888/lab?token=gpu-lab-token
```

Если использовался локальный порт `8889`, открыть:

```text
http://localhost:8889/lab?token=gpu-lab-token
```

Проверка: в терминале с `kubectl port-forward` после открытия страницы должны появиться строки вида `Handling connection for 8888`.

Если в SSH-терминале появляются строки `channel ... open failed: connect failed: Connection refused`, значит браузер уже пытается открыть tunnel, но на VM еще не запущен `kubectl port-forward`. Это не ошибка кластера: сначала дождаться `Forwarding from 127.0.0.1:8888 -> 8888`, потом обновить страницу.

В терминале JupyterLab выполнить именно shell-команду ниже. Она сама запускает Python, поэтому не нужно вводить `import torch` прямо в prompt `#`:

```bash
python - <<'PY'
import torch
print(torch.__version__)
print(torch.cuda.is_available())
print(torch.cuda.get_device_name(0))
print(torch.version.cuda)
PY
```

### Скриншот 6. CUDA доступна в JupyterLab

Сделать скриншот после выполнения Python-команд в терминале JupyterLab. На скриншоте должно быть видно:

- `torch` version `2.5.1+cu124`;
- `torch.cuda.is_available()` возвращает `True`;
- GPU model `Tesla T4`;
- CUDA runtime `12.4`.

После скриншота JupyterLab можно удалить:

```bash
kubectl delete deployment jupyterlab -n gpu-lab
```

## 5. Эксперимент 1: Exclusive GPU

Конфигурация: один pod получает один физический GPU через ресурс `nvidia.com/gpu: 1`.

### Скриншот 7. Логи exclusive benchmark

Выполнить команду:

```bash
kubectl logs -n gpu-lab job/torch-benchmark
```

Сделать скриншот после появления benchmark output. На скриншоте должно быть видно:

- `PyTorch GPU Benchmark - Exclusive mode`;
- `GPU model: Tesla T4`;
- `CUDA version: 12.4`;
- `avg_step_sec`;
- `min_step_sec`;
- `max_step_sec`;
- `peak_cuda_mem_gb`.

Фактические значения:

| Метрика | Значение |
|---------|----------|
| GPU model | Tesla T4 |
| GPU memory | 14.6 GB |
| avg_step_sec | 0.0320-0.0324 |
| min_step_sec | 0.0314-0.0321 |
| max_step_sec | 0.0324-0.0327 |
| peak_cuda_mem_gb | 0.258 |

## 6. Эксперимент 2: Time-slicing

Конфигурация: один физический GPU публикуется как четыре shared-слота `nvidia.com/gpu.shared`.

Перед time-slicing убрать JupyterLab, если он еще запущен после скриншота CUDA. Иначе он может занимать единственную GPU:

```bash
kubectl delete deployment jupyterlab -n gpu-lab --ignore-not-found
kubectl get pods -n gpu-lab
```

### Скриншот 8. Включение time-slicing resource

Выполнить команды:

```bash
NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
kubectl label node "$NODE" nvidia.com/device-plugin.config=t4-timeslicing-4 --overwrite
kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
kubectl describe node "$NODE" | sed -n '/Capacity:/,/Allocatable:/p'
```

Сделать скриншот после последней команды. На скриншоте должно быть видно:

```text
nvidia.com/gpu.shared: 4
```

### Скриншот 9. Запуск четырех parallel job

Выполнить команды:

```bash
cd /home/mrstart151/k8s-gpu-lab

for i in 1 2 3 4; do
  kubectl delete job -n gpu-lab --ignore-not-found "torch-benchmark-ts-$i"
  sed "s/name: torch-benchmark/name: torch-benchmark-ts-$i/" \
    /home/mrstart151/k8s-gpu-lab/gpu-lab/torch-benchmark-shared.yaml | kubectl apply -f -
done

kubectl get pods -n gpu-lab -w
```

Сделать скриншот, когда будут видны четыре pod/job time-slicing. Нормально, если они быстро перейдут в `Completed`.

### Скриншот 10. Логи time-slicing benchmark

Выполнить команду:

```bash
for i in 1 2 3 4; do
  echo "===== torch-benchmark-ts-$i ====="
  kubectl logs -n gpu-lab "job/torch-benchmark-ts-$i"
done
```

Сделать скриншот после появления логов всех четырех job. Если вывод не помещается, сделать несколько скриншотов. На скриншоте должны быть видны `avg_step_sec`, `min_step_sec`, `max_step_sec`, `peak_cuda_mem_gb` для `ts-1`, `ts-2`, `ts-3`, `ts-4`.

Фактические значения:

| Job | avg_step_sec | min_step_sec | max_step_sec | peak_cuda_mem_gb |
|-----|--------------|--------------|--------------|------------------|
| ts-1 | 0.1360 | 0.1032 | 0.1477 | 0.258 |
| ts-2 | 0.1388 | 0.1351 | 0.1448 | 0.258 |
| ts-3 | 0.1317 | 0.0685 | 0.1470 | 0.258 |
| ts-4 | 0.1251 | 0.0298 | 0.1459 | 0.258 |

После скриншотов обязательно вернуть default GPU mode:

```bash
kubectl label node "$NODE" nvidia.com/device-plugin.config- || true
kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
kubectl get node "$NODE" -o jsonpath='{.status.capacity}'
```

Состояние после возврата должно снова содержать:

```text
"nvidia.com/gpu":"1"
```

## 7. Эксперимент 3: MPS

MPS в NVIDIA Kubernetes device plugin имеет экспериментальный статус. В текущем окружении MPS не запустился, потому что device plugin ожидал MPS daemon по `/tmp/nvidia-mps`, но daemon не был поднят driver container'ом автоматически.

Фактическая ошибка:

```text
error waiting for MPS daemon: error checking MPS daemon health: failed to send command to MPS daemon: exit status 1
```

### Скриншот 11. MPS как допустимый failed result

Лучший вариант: сделать скриншот этого раздела отчета, где видна ошибка выше и пояснение, что MPS экспериментальный.

Если преподаватель требует именно терминальный скрин, повторять MPS нужно осторожно:

```bash
NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"
kubectl label node "$NODE" nvidia.com/device-plugin.config=t4-mps-4 --overwrite
kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
kubectl logs -n gpu-operator -l app=nvidia-device-plugin-daemonset --tail=120
```

Сделать скриншот после команды с логами, если в них видна ошибка MPS daemon. После этого вернуть default mode:

```bash
kubectl label node "$NODE" nvidia.com/device-plugin.config- || true
kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
```

## 8. Мониторинг: DCGM, Prometheus, Grafana

Подключение DCGM exporter к Prometheus:

```bash
kubectl label svc -n gpu-operator nvidia-dcgm-exporter monitor=dcgm --overwrite
kubectl apply -f /home/mrstart151/k8s-gpu-lab/gpu-lab/dcgm-servicemonitor.yaml
```

### Скриншот 12. DCGM target в Prometheus

После создания ServiceMonitor подождать 1-2 минуты, пока Prometheus подхватит новую конфигурацию. Затем выполнить команду:

```bash
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/targets?state=any' \
  | jq -r '.data.activeTargets[] | select((.scrapePool|tostring|test("dcgm|gpu-operator|nvidia"; "i")) or (.labels|tostring|test("dcgm|gpu-operator|nvidia"; "i"))) | [.health,.scrapePool,.labels.namespace,.labels.service,.labels.job,.lastError] | @tsv'
```

Сделать скриншот после появления строки DCGM target. На скриншоте должно быть видно:

```text
up    serviceMonitor/gpu-operator/nvidia-dcgm-exporter/0    gpu-operator    nvidia-dcgm-exporter
```

### Скриншот 13. GPU stress для сбора метрик

Выполнить команды:

```bash
kubectl delete job -n gpu-lab --ignore-not-found torch-stress-10min
kubectl apply -f /home/mrstart151/k8s-gpu-lab/gpu-lab/torch-stress-10min.yaml
kubectl logs -n gpu-lab -f job/torch-stress-10min
```

Сделать скриншот во время выполнения job, когда в логах появляются строки `step ... elapsed ... remaining ...`. После завершения можно сделать еще один скриншот с:

```text
Done. Total matmul steps: 15871
```

### Скриншоты 14-18. Grafana / Prometheus GPU metrics

Вариант A: Grafana.

Делать эти скриншоты во время выполнения `torch-stress-10min` или сразу после него. Если stress уже завершился, поставить range `Last 15 minutes`; если графики уже начинают исчезать, поставить `Last 30 minutes`.

Открыть Grafana в отдельном терминале и оставить команду запущенной:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

В браузере открыть:

```text
http://localhost:3000
```

Логин Grafana:

```text
admin
```

Пароль можно получить командой:

```bash
kubectl get secret -n monitoring monitoring-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

В Grafana Explore по очереди ввести метрики и сделать скриншоты графиков за `Last 15 minutes`:

```text
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_MEM_COPY_UTIL
DCGM_FI_DEV_FB_USED
DCGM_FI_DEV_POWER_USAGE
DCGM_FI_DEV_GPU_TEMP
```

Вариант B: terminal screenshots вместо Grafana. Выполнить команды и сделать скриншот после каждой:

```bash
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/query?query=max_over_time%28DCGM_FI_DEV_GPU_UTIL%5B15m%5D%29'
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/query?query=max_over_time%28DCGM_FI_DEV_MEM_COPY_UTIL%5B15m%5D%29'
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/query?query=max_over_time%28DCGM_FI_DEV_FB_USED%5B15m%5D%29'
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/query?query=max_over_time%28DCGM_FI_DEV_POWER_USAGE%5B15m%5D%29'
kubectl get --raw '/api/v1/namespaces/monitoring/services/http:monitoring-kube-prometheus-prometheus:9090/proxy/api/v1/query?query=max_over_time%28DCGM_FI_DEV_GPU_TEMP%5B15m%5D%29'
```

Фактические значения, полученные во время stress job:

| Метрика | Значение | Единицы |
|---------|----------|---------|
| `DCGM_FI_DEV_GPU_UTIL` | 100 | % |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | 24 | % |
| `DCGM_FI_DEV_FB_USED` | 328 | MiB |
| `DCGM_FI_DEV_POWER_USAGE` | 69.079 | W |
| `DCGM_FI_DEV_GPU_TEMP` | 66 | C |

## 9. Сравнение режимов

| Режим | Число pod | Kubernetes resource | avg_step_sec | Пик GPU util | Пик FB used | Комментарий |
|-------|-----------|---------------------|--------------|--------------|-------------|-------------|
| Exclusive | 1 | `nvidia.com/gpu` | 0.0320-0.0324 | до 100% на stress | 328 MiB на stress | Минимальная latency, полный доступ к GPU |
| Time-slicing | 4 | `nvidia.com/gpu.shared` | около 0.133 | высокая суммарная загрузка при parallel jobs | 4 x 0.258 GB CUDA allocation | Latency выросла примерно в 4 раза |
| MPS | не запустился | `nvidia.com/gpu.shared` | нет данных | нет данных | нет данных | MPS daemon отсутствует в текущей driver-container конфигурации |

## 10. Ответы на вопросы

**Почему в exclusive одна job обычно выполняется быстрее?**

В exclusive режиме pod получает физическую GPU полностью в свое распоряжение: нет конкуренции за SM, память и CUDA context. Поэтому latency минимальная.

**Почему в time-slicing можно запустить несколько pod на одной GPU?**

NVIDIA device plugin с включенным `timeSlicing` публикует несколько логических ресурсов `nvidia.com/gpu.shared` поверх одной физической GPU. Kubernetes планирует pod'ы как будто у него есть несколько GPU-слотов, а фактическое разделение выполняется через переключение CUDA context по времени.

**Какие риски появляются при разделении одной GPU между несколькими pod?**

- Нет жесткой аппаратной изоляции памяти.
- Один pod может занять слишком много GPU memory.
- Latency становится менее предсказуемой.
- Один шумный workload может ухудшить производительность соседних pod'ов.

**Чем MPS отличается от time-slicing?**

Time-slicing переключает CUDA contexts целиком. MPS использует общий daemon-сервер, который принимает CUDA-команды от нескольких клиентов и отправляет их на GPU с меньшим overhead. При этом изоляция все равно слабее, чем при выделенном GPU.

**Какой режим выбрать?**

- Одиночная тяжелая задача: `exclusive`.
- Несколько интерактивных JupyterLab или легких inference-сессий: `time-slicing`.
- Batch inference с высоким QPS: `MPS`, если окружение его поддерживает.

## 11. Бонус: Dynamic Resource Allocation (DRA)

### Исследование

По документации NVIDIA DRA Driver for GPUs нужны:

- Kubernetes `v1.34.2+`;
- GPU Operator `v25.10.0+`;
- NVIDIA driver `580+`;
- CDI;
- отключенный NVIDIA device plugin для GPU allocation через DRA.

Текущий стенд подходит по версиям:

```text
Kubernetes: v1.34.8
GPU Operator: v25.10.1
NVIDIA Driver: 580.105.08
DRA API: resource.k8s.io/v1
Chart: nvidia/nvidia-dra-driver-gpu 25.12.0
```

### Скриншот 19. DRA API resources

Выполнить команду:

```bash
kubectl api-resources | grep -E 'deviceclasses|resourceclaims|resourceclaimtemplates|resourceslices'
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

- `deviceclasses`;
- `resourceclaims`;
- `resourceclaimtemplates`;
- `resourceslices`.

### Скриншот 20. NVIDIA DRA Helm chart

Выполнить команду:

```bash
helm search repo nvidia/nvidia-dra-driver-gpu --versions | head
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно chart `nvidia/nvidia-dra-driver-gpu` версии `25.12.0`.

### Скриншот 21. Подготовленные DRA manifests

Выполнить команду:

```bash
ls -l /home/mrstart151/k8s-gpu-lab/dra-values.yaml /home/mrstart151/k8s-gpu-lab/dra-smoke.yaml
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно, что оба файла существуют.

### Практическая попытка DRA

### Скриншот 22. DRA-compatible switch GPU Operator

Выполнить команды:

```bash
kubectl label node epd04l4uglackuqkb8j9 nvidia.com/dra-kubelet-plugin=true --overwrite

helm upgrade --install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --version=v25.10.1 \
  --set driver.enabled=true \
  --set driver.version=580.105.08 \
  --set toolkit.enabled=true \
  --set dcgmExporter.enabled=true \
  --set devicePlugin.enabled=false \
  --set cdi.enabled=true \
  --set cdi.default=true
```

Сделать скриншот после `helm upgrade`. На скриншоте должно быть видно:

- `Release "gpu-operator" has been upgraded`;
- `STATUS: deployed`;
- `REVISION` увеличился.

### Скриншот 23. ClusterPolicy после DRA switch

После переключения выполнить команду:

```bash
kubectl get clusterpolicy cluster-policy -o jsonpath='cdi.enabled={.spec.cdi.enabled}{"\n"}cdi.default={.spec.cdi.default}{"\n"}devicePlugin.enabled={.spec.devicePlugin.enabled}{"\n"}driver.enabled={.spec.driver.enabled}{"\n"}driver.version={.spec.driver.version}{"\n"}'
```

Сделать скриншот после этой команды. На скриншоте должно быть видно, что DRA-compatible mode включен:

```text
cdi.default=true
cdi.enabled=true
devicePlugin.enabled=false
driver.enabled=true
driver.version=580.105.08
```

Практический smoke test был остановлен до установки `nvidia-dra-driver-gpu`: при переключении GPU Operator начал переустанавливать driver container, а внутри pod не работал DNS через `kube-dns` UDP service.

Доказательство ошибки:

```text
W: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/jammy/InRelease  Temporary failure resolving 'archive.ubuntu.com'
W: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/jammy-updates/InRelease  Temporary failure resolving 'archive.ubuntu.com'
W: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/jammy-security/InRelease  Temporary failure resolving 'archive.ubuntu.com'
Could not resolve Linux kernel version
```

Диагностика:

```text
pod -> CoreDNS pod IP UDP: ok
pod -> kube-dns service IP TCP: ok
pod -> kube-dns service IP UDP: timeout
```

Для восстановления базовой лабораторной был выполнен rollback GPU Operator на device plugin mode. Также был исправлен CoreDNS upstream:

Если DRA-compatible switch повторялся во время снятия скриншотов, после фиксации evidence вернуть базовый режим:

```bash
helm upgrade --install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --version=v25.10.1 \
  --set driver.enabled=true \
  --set driver.version=580.105.08 \
  --set toolkit.enabled=true \
  --set dcgmExporter.enabled=true \
  --set devicePlugin.enabled=true \
  --set cdi.enabled=false \
  --set cdi.default=false

kubectl rollout status -n gpu-operator daemonset/nvidia-device-plugin-daemonset --timeout=10m
kubectl exec -n gpu-operator ds/nvidia-driver-daemonset -- nvidia-smi
```

```text
forward . 10.129.0.2
```

Для driver daemonset временно включен `hostNetwork=true` и `dnsPolicy=Default`, чтобы driver container использовал host DNS. После этого driver восстановился:

```text
Post-install sanity check passed.
Loading NVIDIA driver kernel modules...
Starting NVIDIA persistence daemon...
Done, now waiting for signal
```

Финальное состояние после восстановления:

```text
nvidia.com/gpu: 1
nvidia-driver-daemonset: 1/1 Running
nvidia-device-plugin-daemonset: 2/2 Running
```

### Скриншот 24. Возврат базового GPU Operator mode

После rollback выполнить команды:

```bash
kubectl get clusterpolicy cluster-policy -o jsonpath='cdi.enabled={.spec.cdi.enabled}{"\n"}cdi.default={.spec.cdi.default}{"\n"}devicePlugin.enabled={.spec.devicePlugin.enabled}{"\n"}driver.enabled={.spec.driver.enabled}{"\n"}driver.version={.spec.driver.version}{"\n"}'
kubectl get pods -n gpu-operator
kubectl describe node epd04l4uglackuqkb8j9 | sed -n '/Capacity:/,/Allocatable:/p'
```

Сделать скриншот после этих команд. На скриншоте должно быть видно:

- `devicePlugin.enabled=true`;
- `cdi.enabled=false`;
- `nvidia-driver-daemonset` в `Running`;
- `nvidia-device-plugin-daemonset` в `Running`;
- `nvidia.com/gpu: 1`.

### Как завершить DRA smoke test позже

Перед повторной попыткой нужно починить UDP service path до `kube-dns`, либо оставить driver daemonset на `hostNetwork=true` во время DRA switch.

Команды для повторной попытки:

```bash
kubectl label node epd04l4uglackuqkb8j9 nvidia.com/dra-kubelet-plugin=true --overwrite

helm upgrade --install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --version=v25.10.1 \
  --set driver.enabled=true \
  --set driver.version=580.105.08 \
  --set toolkit.enabled=true \
  --set dcgmExporter.enabled=true \
  --set devicePlugin.enabled=false \
  --set cdi.enabled=true \
  --set cdi.default=true

kubectl create namespace nvidia-dra-driver-gpu
kubectl label --overwrite namespace nvidia-dra-driver-gpu pod-security.kubernetes.io/enforce=privileged

helm upgrade -i nvidia-dra-driver-gpu nvidia/nvidia-dra-driver-gpu \
  --version="25.12.0" \
  --namespace nvidia-dra-driver-gpu \
  --set nvidiaDriverRoot=/run/nvidia/driver \
  --set gpuResourcesEnabledOverride=true \
  -f /home/mrstart151/k8s-gpu-lab/dra-values.yaml

kubectl get pods -n nvidia-dra-driver-gpu
kubectl get deviceclass
kubectl get resourceslice
kubectl apply -f /home/mrstart151/k8s-gpu-lab/dra-smoke.yaml
kubectl get resourceclaims -n gpu-dra-smoke
kubectl logs -n gpu-dra-smoke pod/dra-smoke
```

Источники:

- NVIDIA DRA Driver for GPUs: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/dra-intro-install.html
- Kubernetes Dynamic Resource Allocation: https://v1-34.docs.kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/

## 12. Бонус: single-node Slurm cluster

Был развернут single-node Slurm cluster на той же VM.

### Скриншот 25. MUNGE

Выполнить команду:

```bash
munge -n | unmunge
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

```text
STATUS: Success (0)
```

### Скриншот 26. Slurm partition

Выполнить команду:

```bash
sinfo
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

```text
debug*       up   infinite      1   idle epd04l4uglackuqkb8j9
```

### Скриншот 27. Slurm node details

Выполнить команду:

```bash
scontrol show nodes
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

- `NodeName=epd04l4uglackuqkb8j9`;
- `CPUTot=8`;
- `RealMemory=28000`;
- `State=IDLE`;
- `Gres=(null)`.

### Скриншот 28. Slurm `srun`

Выполнить команду:

```bash
srun -N1 -n1 hostname
```

Сделать скриншот после появления вывода:

```text
epd04l4uglackuqkb8j9
```

### Скриншот 29. Slurm batch output

Выполнить команду:

```bash
cat /home/mrstart151/k8s-gpu-lab/slurm-batch-1.out
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно:

- hostname `epd04l4uglackuqkb8j9`;
- `CPU(s): 8`;
- `Model name: Intel Xeon Processor (Icelake)`.

### Скриншот 30. Slurm queue

Выполнить команду:

```bash
squeue
```

Сделать скриншот после появления вывода. Очередь может быть пустой, это нормально.

### Скриншот 31. Почему Slurm без GPU GRES

Выполнить команды:

```bash
which nvidia-smi || true
ls -l /dev/nvidia0 /dev/nvidiactl /dev/nvidia-uvm
```

Сделать скриншот после появления вывода. На скриншоте должно быть видно, что на host нет `nvidia-smi` и нет `/dev/nvidia*`.

Вывод: GPU GRES в Slurm не включен, потому что драйвером управляет NVIDIA GPU Operator внутри Kubernetes driver container. Slurm ожидает host-visible GPU device files, а текущая лабораторная использует Kubernetes device plugin.

Источники:

- Slurm Quick Start Admin Guide: https://slurm.schedmd.com/quickstart_admin.html
- Slurm Quick Start User Guide: https://slurm.schedmd.com/quickstart.html
