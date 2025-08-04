---
title: 简记immich初体验
published: 2024-08-18
tags:
  - Software
  - AIGC
category: Zheteng
draft: false
---

从大概半年前我就在关注immich了，不过因为服务器架构频繁变动，因此没有去动手部署。目前已经基本定型（其实是没精力折腾了），因此开始部署一些“家庭必备”小服务。  

### GPU Support
PVE，IOMMU，禁用nouveau，直通显卡，ubuntu（其他发行版也可），Nvidia-driver，CUDA，CUDNN，老生常谈的事情了，这里不再赘述  
随后需要为docker启用GPU支持，docker安装，nvidia-container-toolkit，随手丢个链接：[https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)  
需要时请查阅最新版文档，旧版本不一定生效（痛骂NoVideo）

### Docker Compose

#### 懒狗快捷指南：

[docker-compose.yml](#section1)  
[hwaccel.ml.yml](#section2)  
[hwaccel.transcoding.yml](#section3)  
[.env](#section4)

#### 具体实现：
接下来的四个文件是按照官网说明编写的，但运行时出现了问题，**不要直接复制粘贴！！！**  

`docker-compose.yml`

```yaml
#
# WARNING: Make sure to use the docker-compose.yml of the current release:
#
# https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
#
# The compose file on main may not be compatible with the latest release.
#

name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    extends:
      file: hwaccel.transcoding.yml
      service: nvenc # Set to 'nvenc' for Nvidia GPU-based hardware transcoding
      #   service: cpu # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
    volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - 2283:3001
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich_machine_learning
    # For hardware acceleration, add one of -[armnn, cuda, openvino] to the image tag.
    # Example tag: ${IMMICH_VERSION:-release}-cuda
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-cuda # Adding -cuda to enable CUDA support
    extends: # uncomment this section for hardware acceleration - see https://immich.app/docs/features/ml-hardware-acceleration
      file: hwaccel.ml.yml
      service: cuda # Set to 'cuda' for Nvidia GPU-based machine learning acceleration
      #   service: cpu # set to one of [armnn, cuda, openvino, openvino-wsl] for accelerated inference - use the `-wsl` version for WSL2 where applicable
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false

  redis:
    container_name: immich_redis
    image: docker.io/redis:6.2-alpine@sha256:e3b17ba9479deec4b7d1eeec1548a253acc5374d68d3b27937fcfe4df8d18c7e
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:90724186f0a3517cf6914295b5ab410db9ce23190a2d9d0b9dd6463e3fa298f0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      # Do not edit the next line. If you want to change the database storage location on your system, edit the value of DB_DATA_LOCATION in the .env file
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready --dbname='${DB_DATABASE_NAME}' --username='${DB_USERNAME}' || exit 1; Chksum="$$(psql --dbname='${DB_DATABASE_NAME}' --username='${DB_USERNAME}' --tuples-only --no-align --command='SELECT COALESCE(SUM(checksum_failures), 0) FROM pg_stat_database')"; echo "checksum failure count is $$Chksum"; [ "$$Chksum" = '0' ] || exit 1
      interval: 5m
      start_interval: 30s
      start_period: 5m
    command: ["postgres", "-c", "shared_preload_libraries=vectors.so", "-c", 'search_path="$$user", public, vectors', "-c", "logging_collector=on", "-c", "max_wal_size=2GB", "-c", "shared_buffers=512MB", "-c", "wal_compression=on"]
    restart: always

volumes:
  model-cache:
```

<h6 id="section2">hwaccel.ml.yml</h6>

```yaml
# Configurations for hardware-accelerated machine learning

# If using Unraid or another platform that doesn't allow multiple Compose files,
# you can inline the config for a backend by copying its contents
# into the immich-machine-learning service in the docker-compose.yml file.

# See https://immich.app/docs/features/ml-hardware-acceleration for info on usage.

services:
  armnn:
    devices:
      - /dev/mali0:/dev/mali0
    volumes:
      - /lib/firmware/mali_csffw.bin:/lib/firmware/mali_csffw.bin:ro # Mali firmware for your chipset (not always required depending on the driver)
      - /usr/lib/libmali.so:/usr/lib/libmali.so:ro # Mali driver for your chipset (always required)

  cpu: {}

  cuda:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities:
                - gpu

  openvino:
    device_cgroup_rules:
      - 'c 189:* rmw'
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /dev/bus/usb:/dev/bus/usb

  openvino-wsl:
    devices:
      - /dev/dri:/dev/dri
      - /dev/dxg:/dev/dxg
    volumes:
      - /dev/bus/usb:/dev/bus/usb
      - /usr/lib/wsl:/usr/lib/wsl
```

<h6 id="section3">hwaccel.transcoding.yml</h6>

```yaml
# Configurations for hardware-accelerated transcoding

# If using Unraid or another platform that doesn't allow multiple Compose files,
# you can inline the config for a backend by copying its contents
# into the immich-microservices service in the docker-compose.yml file.

# See https://immich.app/docs/features/hardware-transcoding for more info on using hardware transcoding.

services:
  cpu: {}

  nvenc:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities:
                - gpu
                - compute
                - video

  quicksync:
    devices:
      - /dev/dri:/dev/dri

  rkmpp:
    security_opt: # enables full access to /sys and /proc, still far better than privileged: true
      - systempaths=unconfined
      - apparmor=unconfined
    group_add:
      - video
    devices:
      - /dev/rga:/dev/rga
      - /dev/dri:/dev/dri
      - /dev/dma_heap:/dev/dma_heap
      - /dev/mpp_service:/dev/mpp_service
      #- /dev/mali0:/dev/mali0 # only required to enable OpenCL-accelerated HDR -> SDR tonemapping
    volumes:
      #- /etc/OpenCL:/etc/OpenCL:ro # only required to enable OpenCL-accelerated HDR -> SDR tonemapping
      #- /usr/lib/aarch64-linux-gnu/libmali.so.1:/usr/lib/aarch64-linux-gnu/libmali.so.1:ro # only required to enable OpenCL-accelerated HDR -> SDR tonemapping

  vaapi:
    devices:
      - /dev/dri:/dev/dri

  vaapi-wsl: # use this for VAAPI if you're running Immich in WSL2
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /usr/lib/wsl:/usr/lib/wsl
    environment:
      - LD_LIBRARY_PATH=/usr/lib/wsl/lib
      - LIBVA_DRIVER_NAME=d3d12
```

<h6 id="section4">.env</h6>

```
# You can find documentation for all the supported env variables at https://immich.app/docs/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=/AppData/immich/library
# The location where your database files are stored
DB_DATA_LOCATION=/AppData/immich/postgres

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
# TZ=Etc/UTC

# The Immich version to use. You can pin this to a specific version like "v1.71.0"
IMMICH_VERSION=release

# Connection secret for postgres. You should change it to a random password
# Please use only the characters `A-Za-z0-9`, without special characters or spaces
DB_PASSWORD=postgres

# The values below this line do not need to be changed
###################################################################################
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

---

通过查看日志发现了问题：

```
                             RuntimeError:
                             /onnxruntime_src/onnxruntime/core/providers/cuda/cu
                             da_call.cc:123 std::conditional_t<THRW, void,
                             onnxruntime::common::Status>
                             onnxruntime::CudaCall(ERRTYPE, const char*, const
                             char*, ERRTYPE, const char*, const char*, int)
                             [with ERRTYPE = cudaError; bool THRW = true;
                             std::conditional_t<THRW, void, common::Status> =
                             void]
                             /onnxruntime_src/onnxruntime/core/providers/cuda/cu
                             da_call.cc:116 std::conditional_t<THRW, void,
                             onnxruntime::common::Status>
                             onnxruntime::CudaCall(ERRTYPE, const char*, const
                             char*, ERRTYPE, const char*, const char*, int)
                             [with ERRTYPE = cudaError; bool THRW = true;
                             std::conditional_t<THRW, void, common::Status> =
                             void] CUDA failure 100: no CUDA-capable device is
                             detected ; GPU=30355 ; hostname=f0cdef844009 ;
                             file=/onnxruntime_src/onnxruntime/core/providers/cu
                             da/cuda_execution_provider.cc ; line=280 ;
                             expr=cudaSetDevice(info_.device_id);



                             The above exception was the direct cause of the
                             following exception:

                             ╭─────── Traceback (most recent call last) ───────╮
                             │ /usr/src/app/main.py:152 in predict             │
                             │                                                 │
                             │   149 │   │   inputs = text                     │
                             │   150 │   else:                                 │
                             │   151 │   │   raise HTTPException(400, "Either  │
                             │ ❱ 152 │   response = await run_inference(inputs │
                             │   153 │   return ORJSONResponse(response)       │
                             │   154                                           │
                             │   155                                           │
                             │                                                 │
                             │ /usr/src/app/main.py:175 in run_inference       │
                             │                                                 │
                             │   172 │   │   response[entry["task"]] = output  │
                             │   173 │                                         │
                             │   174 │   without_deps, with_deps = entries     │
                             │ ❱ 175 │   await asyncio.gather(*[_run_inference │
                             │   176 │   if with_deps:                         │
                             │   177 │   │   await asyncio.gather(*[_run_infer │
                             │   178 │   if isinstance(payload, Image):        │
                             │                                                 │
                             │ /usr/src/app/main.py:169 in _run_inference      │
                             │                                                 │
                             │   166 │   │   │   except KeyError:              │
                             │   167 │   │   │   │   message = f"Task {entry[' │
                             │       output of {dep}"                          │
                             │   168 │   │   │   │   raise HTTPException(400,  │
                             │ ❱ 169 │   │   model = await load(model)         │
                             │   170 │   │   output = await run(model.predict, │
                             │   171 │   │   outputs[model.identity] = output  │
                             │   172 │   │   response[entry["task"]] = output  │
                             │                                                 │
                             │ /usr/src/app/main.py:213 in load                │
                             │                                                 │
                             │   210 │   │   return model                      │
                             │   211 │                                         │
                             │   212 │   try:                                  │
                             │ ❱ 213 │   │   return await run(_load, model)    │
                             │   214 │   except (OSError, InvalidProtobuf, Bad │
                             │   215 │   │   log.warning(f"Failed to load {mod │
                             │       '{model.model_name}'. Clearing cache.")   │
                             │   216 │   │   model.clear_cache()               │
                             │                                                 │
                             │ /usr/src/app/main.py:188 in run                 │
                             │                                                 │
                             │   185 │   if thread_pool is None:               │
                             │   186 │   │   return func(*args, **kwargs)      │
                             │   187 │   partial_func = partial(func, *args, * │
                             │ ❱ 188 │   return await asyncio.get_running_loop │
                             │   189                                           │
                             │   190                                           │
                             │   191 async def load(model: InferenceModel) ->  │
                             │                                                 │
                             │ /usr/local/lib/python3.11/concurrent/futures/th │
                             │ read.py:58 in run                               │
                             │                                                 │
                             │ /usr/src/app/main.py:200 in _load               │
                             │                                                 │
                             │   197 │   │   │   raise HTTPException(500, f"Fa │
                             │   198 │   │   with lock:                        │
                             │   199 │   │   │   try:                          │
                             │ ❱ 200 │   │   │   │   model.load()              │
                             │   201 │   │   │   except FileNotFoundError as e │
                             │   202 │   │   │   │   if model.model_format ==  │
                             │   203 │   │   │   │   │   raise e               │
                             │                                                 │
                             │ /usr/src/app/models/base.py:53 in load          │
                             │                                                 │
                             │    50 │   │   self.download()                   │
                             │    51 │   │   attempt = f"Attempt #{self.load_a │
                             │       else "Loading"                            │
                             │    52 │   │   log.info(f"{attempt} {self.model_ │
                             │       '{self.model_name}' to memory")           │
                             │ ❱  53 │   │   self.session = self._load()       │
                             │    54 │   │   self.loaded = True                │
                             │    55 │                                         │
                             │    56 │   def predict(self, *inputs: Any, **mod │
                             │                                                 │
                             │ /usr/src/app/models/clip/visual.py:62 in _load  │
                             │                                                 │
                             │   59 │   │   self.mean = np.array(self.preproce │
                             │   60 │   │   self.std = np.array(self.preproces │
                             │   61 │   │                                      │
                             │ ❱ 62 │   │   return super()._load()             │
                             │   63 │                                          │
                             │   64 │   def transform(self, image: Image.Image │
                             │   65 │   │   image = resize_pil(image, self.siz │
                             │                                                 │
                             │ /usr/src/app/models/base.py:79 in _load         │
                             │                                                 │
                             │    76 │   │   )                                 │
                             │    77 │                                         │
                             │    78 │   def _load(self) -> ModelSession:      │
                             │ ❱  79 │   │   return self._make_session(self.mo │
                             │    80 │                                         │
                             │    81 │   def clear_cache(self) -> None:        │
                             │    82 │   │   if not self.cache_dir.exists():   │
                             │                                                 │
                             │ /usr/src/app/models/base.py:111 in              │
                             │ _make_session                                   │
                             │                                                 │
                             │   108 │   │   │   case ".armnn":                │
                             │   109 │   │   │   │   session: ModelSession = A │
                             │   110 │   │   │   case ".onnx":                 │
                             │ ❱ 111 │   │   │   │   session = OrtSession(mode │
                             │   112 │   │   │   case _:                       │
                             │   113 │   │   │   │   raise ValueError(f"Unsupp │
                             │   114 │   │   return session                    │
                             │                                                 │
                             │ /usr/src/app/sessions/ort.py:28 in __init__     │
                             │                                                 │
                             │    25 │   │   self.providers = providers if pro │
                             │    26 │   │   self.provider_options = provider_ │
                             │       self._provider_options_default            │
                             │    27 │   │   self.sess_options = sess_options  │
                             │       self._sess_options_default                │
                             │ ❱  28 │   │   self.session = ort.InferenceSessi │
                             │    29 │   │   │   self.model_path.as_posix(),   │
                             │    30 │   │   │   providers=self.providers,     │
                             │    31 │   │   │   provider_options=self.provide │
                             │                                                 │
                             │ /opt/venv/lib/python3.11/site-packages/onnxrunt │
                             │ ime/capi/onnxruntime_inference_collection.py:43 │
                             │ 2 in __init__                                   │
                             │                                                 │
                             │    429 │   │   │   │   │   self.disable_fallbac │
                             │    430 │   │   │   │   │   return               │
                             │    431 │   │   │   │   except Exception as fall │
                             │ ❱  432 │   │   │   │   │   raise fallback_error │
                             │    433 │   │   │   # Fallback is disabled. Rais │
                             │    434 │   │   │   raise e                      │
                             │    435                                          │
                             │                                                 │
                             │ /opt/venv/lib/python3.11/site-packages/onnxrunt │
                             │ ime/capi/onnxruntime_inference_collection.py:42 │
                             │ 7 in __init__                                   │
                             │                                                 │
                             │    424 │   │   │   │   │   print(f"EP Error {e} │
                             │    425 │   │   │   │   │   print(f"Falling back │
                             │    426 │   │   │   │   │   print("************* │
                             │ ❱  427 │   │   │   │   │   self._create_inferen │
                             │    428 │   │   │   │   │   # Fallback only once │
                             │    429 │   │   │   │   │   self.disable_fallbac │
                             │    430 │   │   │   │   │   return               │
                             │                                                 │
                             │ /opt/venv/lib/python3.11/site-packages/onnxrunt │
                             │ ime/capi/onnxruntime_inference_collection.py:48 │
                             │ 3 in _create_inference_session                  │
                             │                                                 │
                             │    480 │   │   │   disabled_optimizers = set(di │
                             │    481 │   │                                    │
                             │    482 │   │   # initialize the C++ InferenceSe │
                             │ ❱  483 │   │   sess.initialize_session(provider │
                             │    484 │   │                                    │
                             │    485 │   │   self._sess = sess                │
                             │    486 │   │   self._sess_options = self._sess. │
                             ╰─────────────────────────────────────────────────╯
                             RuntimeError:
                             /onnxruntime_src/onnxruntime/core/providers/cuda/cu
                             da_call.cc:123 std::conditional_t<THRW, void,
                             onnxruntime::common::Status>
                             onnxruntime::CudaCall(ERRTYPE, const char*, const
                             char*, ERRTYPE, const char*, const char*, int)
                             [with ERRTYPE = cudaError; bool THRW = true;
                             std::conditional_t<THRW, void, common::Status> =
                             void]
                             /onnxruntime_src/onnxruntime/core/providers/cuda/cu
                             da_call.cc:116 std::conditional_t<THRW, void,
                             onnxruntime::common::Status>
                             onnxruntime::CudaCall(ERRTYPE, const char*, const
                             char*, ERRTYPE, const char*, const char*, int)
                             [with ERRTYPE = cudaError; bool THRW = true;
                             std::conditional_t<THRW, void, common::Status> =
                             void] CUDA failure 100: no CUDA-capable device is
                             detected ; GPU=30355 ; hostname=f0cdef844009 ;
                             file=/onnxruntime_src/onnxruntime/core/providers/cu
                             da/cuda_execution_provider.cc ; line=280 ;
                             expr=cudaSetDevice(info_.device_id);


[08/18/24 10:37:48] INFO     Shutting down due to inactivity.
[08/18/24 10:37:48] INFO     Shutting down
[08/18/24 10:37:48] INFO     Waiting for application shutdown.
[08/18/24 10:37:49] INFO     Application shutdown complete.
[08/18/24 10:37:49] INFO     Finished server process [51638]
[08/18/24 10:37:49] ERROR    Worker (pid:51638) was sent SIGINT!
[08/18/24 10:37:49] INFO     Booting worker with pid: 56010
[08/18/24 10:37:53] INFO     Started server process [56010]
[08/18/24 10:37:53] INFO     Waiting for application startup.
[08/18/24 10:37:53] INFO     Created in-memory cache with unloading after 300s
                             of inactivity.
[08/18/24 10:37:53] INFO     Initialized request thread pool with 16 threads.
[08/18/24 10:37:53] INFO     Application startup complete.
```

根据报错可以发现：没有支持CUDA的设备，即容器的GPU支持配置有误  
因此修改配置文件：

<h6 id="section1">docker-compose.yml</h6>

```yaml
# Configurations for hardware-accelerated transcoding

# If using Unraid or another platform that doesn't allow multiple Compose files,
# you can inline the config for a backend by copying its contents
# into the immich-microservices service in the docker-compose.yml file.

# See https://immich.app/docs/features/hardware-transcoding for more info on using hardware transcoding.

services:
  cpu: {}

  nvenc:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities:
                - gpu
                - compute
                - video

  quicksync:
    devices:
      - /dev/dri:/dev/dri

  rkmpp:
    security_opt: # enables full access to /sys and /proc, still far better than privileged: true
      - systempaths=unconfined
      - apparmor=unconfined
    group_add:
      - video
    devices:
      - /dev/rga:/dev/rga
      - /dev/dri:/dev/dri
      - /dev/dma_heap:/dev/dma_heap
      - /dev/mpp_service:/dev/mpp_service
      #- /dev/mali0:/dev/mali0 # only required to enable OpenCL-accelerated HDR -> SDR tonemapping
    volumes:
      #- /etc/OpenCL:/etc/OpenCL:ro # only required to enable OpenCL-accelerated HDR -> SDR tonemapping
      #- /usr/lib/aarch64-linux-gnu/libmali.so.1:/usr/lib/aarch64-linux-gnu/libmali.so.1:ro # only required to enable OpenCL-accelerated HDR -> SDR tonemapping

  vaapi:
    devices:
      - /dev/dri:/dev/dri

  vaapi-wsl: # use this for VAAPI if you're running Immich in WSL2
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /usr/lib/wsl:/usr/lib/wsl
    environment:
      - LD_LIBRARY_PATH=/usr/lib/wsl/lib
      - LIBVA_DRIVER_NAME=d3d12
root@ai-p100:/AppData/immich/immich-app# cat .env
# You can find documentation for all the supported env variables at https://immich.app/docs/install/environment-variables

# The location where your uploaded files are stored
UPLOAD_LOCATION=/AppData/immich/library
# The location where your database files are stored
DB_DATA_LOCATION=/AppData/immich/postgres

# To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
# TZ=Etc/UTC

# The Immich version to use. You can pin this to a specific version like "v1.71.0"
IMMICH_VERSION=release

# Connection secret for postgres. You should change it to a random password
# Please use only the characters `A-Za-z0-9`, without special characters or spaces
DB_PASSWORD=postgres

# The values below this line do not need to be changed
###################################################################################
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
root@ai-p100:/AppData/immich/immich-app# cat docker-compose.yml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    extends:
      file: hwaccel.transcoding.yml
      service: nvenc
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    ports:
      - 2283:3001
    depends_on:
      - redis
      - database
    restart: always
    healthcheck:
      disable: false
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu, compute, video]

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-cuda
    extends:
      file: hwaccel.ml.yml
      service: cuda
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  redis:
    container_name: immich_redis
    image: docker.io/redis:6.2-alpine@sha256:e3b17ba9479deec4b7d1eeec1548a253acc5374d68d3b27937fcfe4df8d18c7e
    healthcheck:
      test: redis-cli ping || exit 1
    restart: always

  database:
    container_name: immich_postgres
    image: docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0@sha256:90724186f0a3517cf6914295b5ab410db9ce23190a2d9d0b9dd6463e3fa298f0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready --dbname='${DB_DATABASE_NAME}' --username='${DB_USERNAME}' || exit 1; Chksum="$$(psql --dbname='${DB_DATABASE_NAME}' --username='${DB_USERNAME}' --tuples-only --no-align --command='SELECT COALESCE(SUM(checksum_failures), 0) FROM pg_stat_database')"; echo "checksum failure count is $$Chksum"; [ "$$Chksum" = '0' ] || exit 1
      interval: 5m
      start_interval: 30s
      start_period: 5m
    command: ["postgres", "-c", "shared_preload_libraries=vectors.so", "-c", 'search_path="$$user", public, vectors', "-c", "logging_collector=on", "-c", "max_wal_size=2GB", "-c", "shared_buffers=512MB", "-c", "wal_compression=on"]
    restart: always

volumes:
  model-cache:
```

其中区别就是：添加了以下字段：

```yaml
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### 具体配置

首先是照片上传，移动端几乎全平台支持，直接下载App然后授权访问相册就可以了  
PC稍微麻烦些，需要安装node.js后安装immich-cli，具体请查阅文档  
immich-cli需要node.js版本在20.0以上，所以如果发行版的软件源较为老旧，请换更新的`nodesource`源  

然后是备份，不要直接备份数据库目录，具体的先挖个坑，暂时懒得写QwQ