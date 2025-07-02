This shows the process of how to compile whole Ray( not just python part) on Linux.
## Steps
### 1. Create Virtual Env
```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip wheel # upgrade wheel
```
### 2. Preparation
```bash
sudo apt-get update
sudo apt-get install -y build-essential curl clang-12 pkg-config psmisc unzip

# Install Bazelisk.
ci/env/install-bazel.sh

# Install node version manager and node 14
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 14
nvm use 14
```

### Building
我自己實測用8core大概要24-25G的RAM, 4Core大概就12G, 所以如果會crash就自己抓一下,不要給太多CPU.
>[!Tip] 
>If your machine is running out of memory during the build or the build is causing other programs to crash, try adding the following line to `~/.bazelrc`:
`build --local_ram_resources=HOST_RAM*.5 --local_cpu_resources=4`
The `build --disk_cache=~/bazel-cache` option can be useful to speed up repeated builds too.


```bash
cd ray/python/
pip install -r requirements.txt
pip install -e . --verbose
```

### Build logs
```log
❯ pip install -e . --verbose
Using pip 25.1.1 from /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/pip (python 3.10)
Obtaining file:///home/owenowenisme/ray/python
  Running command python setup.py egg_info
  /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/setuptools/installer.py:27: SetuptoolsDeprecationWarning: setuptools.installer is deprecated. Requirements should be satisfied by a PEP 517 installer.
    warnings.warn(
  running egg_info
  creating /tmp/pip-pip-egg-info-7867k9up/ray.egg-info
  writing /tmp/pip-pip-egg-info-7867k9up/ray.egg-info/PKG-INFO
  writing dependency_links to /tmp/pip-pip-egg-info-7867k9up/ray.egg-info/dependency_links.txt
  writing entry points to /tmp/pip-pip-egg-info-7867k9up/ray.egg-info/entry_points.txt
  writing requirements to /tmp/pip-pip-egg-info-7867k9up/ray.egg-info/requires.txt
  writing top-level names to /tmp/pip-pip-egg-info-7867k9up/ray.egg-info/top_level.txt
  writing manifest file '/tmp/pip-pip-egg-info-7867k9up/ray.egg-info/SOURCES.txt'
  reading manifest file '/tmp/pip-pip-egg-info-7867k9up/ray.egg-info/SOURCES.txt'
  reading manifest template 'MANIFEST.in'
  writing manifest file '/tmp/pip-pip-egg-info-7867k9up/ray.egg-info/SOURCES.txt'
  Preparing metadata (setup.py) ... done
Requirement already satisfied: click>=7.0 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (8.2.1)
Requirement already satisfied: filelock in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (3.18.0)
Requirement already satisfied: jsonschema in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (4.24.0)
Requirement already satisfied: msgpack<2.0.0,>=1.0.0 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (1.1.1)
Requirement already satisfied: packaging in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (25.0)
Requirement already satisfied: protobuf!=3.19.5,>=3.15.3 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (5.29.5)
Requirement already satisfied: pyyaml in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (6.0.2)
Requirement already satisfied: requests in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from ray==3.0.0.dev0) (2.32.4)
Requirement already satisfied: attrs>=22.2.0 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from jsonschema->ray==3.0.0.dev0) (25.3.0)
Requirement already satisfied: jsonschema-specifications>=2023.03.6 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from jsonschema->ray==3.0.0.dev0) (2025.4.1)
Requirement already satisfied: referencing>=0.28.4 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from jsonschema->ray==3.0.0.dev0) (0.36.2)
Requirement already satisfied: rpds-py>=0.7.1 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from jsonschema->ray==3.0.0.dev0) (0.25.1)
Requirement already satisfied: typing-extensions>=4.4.0 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from referencing>=0.28.4->jsonschema->ray==3.0.0.dev0) (4.14.0)
Requirement already satisfied: charset_normalizer<4,>=2 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from requests->ray==3.0.0.dev0) (3.4.2)
Requirement already satisfied: idna<4,>=2.5 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from requests->ray==3.0.0.dev0) (3.10)
Requirement already satisfied: urllib3<3,>=1.21.1 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from requests->ray==3.0.0.dev0) (2.4.0)
Requirement already satisfied: certifi>=2017.4.17 in /home/owenowenisme/ray/.venv/lib/python3.10/site-packages (from requests->ray==3.0.0.dev0) (2025.6.15)
Installing collected packages: ray
  DEPRECATION: Legacy editable install of ray==3.0.0.dev0 from file:///home/owenowenisme/ray/python (setup.py develop) is deprecated. pip 25.3 will enforce this behaviour change. A possible replacement is to add a pyproject.toml or enable --use-pep517, and use setuptools >= 64. If the resulting installation is not behaving as expected, try using --config-settings editable_mode=compat. Please consult the setuptools documentation for more information. Discussion can be found at https://github.com/pypa/pip/issues/11457
  Running setup.py develop for ray
    Running command python setup.py develop
    /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/setuptools/installer.py:27: SetuptoolsDeprecationWarning: setuptools.installer is deprecated. Requirements should be satisfied by a PEP 517 installer.
      warnings.warn(
    running develop
    /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/setuptools/command/easy_install.py:158: EasyInstallDeprecationWarning: easy_install command is deprecated. Use build and pip and other standards-based tools.
      warnings.warn(
    /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/setuptools/command/install.py:34: SetuptoolsDeprecationWarning: setup.py install is deprecated. Use build and pip and other standards-based tools.
      warnings.warn(
    running egg_info
    writing ray.egg-info/PKG-INFO
    writing dependency_links to ray.egg-info/dependency_links.txt
    writing entry points to ray.egg-info/entry_points.txt
    writing requirements to ray.egg-info/requires.txt
    writing top-level names to ray.egg-info/top_level.txt
    reading manifest file 'ray.egg-info/SOURCES.txt'
    reading manifest template 'MANIFEST.in'
    writing manifest file 'ray.egg-info/SOURCES.txt'
    running build_ext
    WARNING: Target directory /home/owenowenisme/ray/python/ray/thirdparty_files/psutil-7.0.0.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/thirdparty_files/colorama already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/thirdparty_files/psutil already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/thirdparty_files/colorama-0.4.6.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiohttp already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiohappyeyeballs-2.6.1.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/frozenlist-1.7.0.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiosignal-1.3.2.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/yarl-1.20.1.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiosignal already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/attrs already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/yarl already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/async_timeout already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/idna-3.10.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/propcache-0.3.2.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/__pycache__ already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/propcache already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/typing_extensions-4.14.0.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/async_timeout-5.0.1.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/idna already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/typing_extensions.py already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiohappyeyeballs already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/multidict already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/frozenlist already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/multidict-6.4.4.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/attr already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/aiohttp-3.12.13.dist-info already exists. Specify --upgrade to force replacement.
    WARNING: Target directory /home/owenowenisme/ray/python/ray/_private/runtime_env/agent/thirdparty_files/attrs-25.3.0.dist-info already exists. Specify --upgrade to force replacement.
    Starting local Bazel server and connecting to it...
    Loading:
    DEBUG: /home/owenowenisme/ray/bazel/ray_deps_setup.bzl:67:14: No implicit mirrors used for com_github_spdlog because urls were explicitly provided
    DEBUG: /home/owenowenisme/ray/bazel/ray_deps_setup.bzl:67:14: No implicit mirrors used for com_github_opentelemetry_proto because urls were explicitly provided
    DEBUG: /home/owenowenisme/ray/bazel/ray_deps_setup.bzl:67:14: No implicit mirrors used for com_google_absl because urls were explicitly provided
    Loading:
    Loading:
    Loading: 0 packages loaded
    Analyzing: 2 targets (2 packages loaded, 0 targets configured)
    Analyzing: 2 targets (63 packages loaded, 3111 targets configured)
    DEBUG: /home/owenowenisme/ray/bazel/ray.bzl:208:10: [<generated file external/jemalloc/libjemalloc/lib/libjemalloc.so>]
    Analyzing: 2 targets (162 packages loaded, 11262 targets configured)
    Analyzing: 2 targets (234 packages loaded, 17805 targets configured)
    INFO: Analyzed 2 targets (242 packages loaded, 25123 targets configured).
     checking cached actions
    INFO: Found 2 targets...
    INFO: Deleting stale sandbox base /home/owenowenisme/.cache/bazel/_bazel_owenowenisme/b8fbf63097444d6c7ab977acc033b9f6/sandbox
    [42 / 2,813] [Prepa] BazelWorkspaceStatusAction stable-status.txt
    [2,040 / 4,515] checking cached actions
    [7,912 / 8,118] [Scann] Compiling src/ray/core_worker/common.cc
    [7,981 / 8,118] Compiling src/ray/core_worker/context.cc; 1s linux-sandbox ... (16 actions, 4 running)
    [7,981 / 8,118] Compiling src/ray/core_worker/context.cc; 12s linux-sandbox ... (16 actions, 4 running)
    [7,981 / 8,118] Compiling src/ray/core_worker/context.cc; 30s linux-sandbox ... (16 actions, 5 running)
    [7,983 / 8,118] Compiling src/ray/core_worker/context.cc; 31s linux-sandbox ... (16 actions, 4 running)
    [7,984 / 8,118] Compiling src/ray/core_worker/context.cc; 32s linux-sandbox ... (16 actions, 4 running)
    [7,984 / 8,118] Compiling src/ray/core_worker/context.cc; 34s linux-sandbox ... (16 actions, 5 running)
    [7,985 / 8,118] Compiling src/ray/core_worker/context.cc; 36s linux-sandbox ... (16 actions, 4 running)
    [7,985 / 8,118] Compiling src/ray/core_worker/context.cc; 37s linux-sandbox ... (16 actions, 5 running)
    [7,986 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/gcs_client.cc; 39s ... (16 actions, 4 running)
    [7,986 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/gcs_client.cc; 48s ... (16 actions, 4 running)
    [7,986 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/global_state_accessor.cc; 59s ... (16 actions, 5 running)
    [7,987 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/global_state_accessor.cc; 60s ... (16 actions, 4 running)
    [7,987 / 8,118] [Sched] Compiling src/ray/core_worker/task_event_buffer.cc; 65s ... (16 actions, 5 running)
    [7,988 / 8,118] [Sched] Compiling src/ray/core_worker/task_event_buffer.cc; 67s ... (16 actions, 4 running)
    [7,989 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/pubsub_handler.cc; 68s ... (16 actions, 4 running)
    [7,990 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/usage_stats_client.cc; 69s ... (16 actions, 4 running)
    [7,990 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/usage_stats_client.cc; 78s ... (16 actions, 4 running)
    [7,990 / 8,118] [Sched] Compiling src/ray/core_worker/profile_event.cc; 102s ... (16 actions, 5 running)
    [7,991 / 8,118] [Sched] Compiling src/ray/core_worker/profile_event.cc; 104s ... (16 actions, 4 running)
    [7,991 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/pubsub_handler.cc; 105s ... (16 actions, 5 running)
    [7,992 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/pubsub_handler.cc; 107s ... (16 actions, 4 running)
    [7,992 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client_pool.cc; 108s ... (16 actions, 5 running)
    [7,993 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client_pool.cc; 110s ... (16 actions, 4 running)
    [7,993 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client_pool.cc; 120s ... (16 actions, 4 running)
    [7,993 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/accessor.cc; 96s ... (16 actions, 5 running)
    [7,994 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/accessor.cc; 97s ... (16 actions, 4 running)
    [7,994 / 8,118] [Sched] Compiling src/ray/core_worker/transport/out_of_order_actor_scheduling_queue.cc; 100s ... (16 actions, 5 running)
    [7,995 / 8,118] [Sched] Compiling src/ray/core_worker/transport/out_of_order_actor_scheduling_queue.cc; 101s ... (16 actions, 4 running)
    [7,995 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/gcs_client.cc; 108s ... (16 actions, 5 running)
    [7,996 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/gcs_client.cc; 109s ... (16 actions, 4 running)
    [7,996 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/gcs_client.cc; 119s ... (16 actions, 4 running)
    [7,996 / 8,118] [Sched] Compiling src/ray/object_manager/ownership_object_directory.cc; 130s ... (16 actions, 5 running)
    [7,997 / 8,118] [Sched] Compiling src/ray/object_manager/ownership_object_directory.cc; 131s ... (16 actions, 4 running)
    [7,997 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/local_resource_manager.cc; 116s ... (16 actions, 5 running)
    [7,998 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/local_resource_manager.cc; 118s ... (16 actions, 4 running)
    [7,998 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/local_resource_manager.cc; 128s ... (16 actions, 4 running)
    [7,998 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_manager.cc; 140s ... (16 actions, 5 running)
    [7,999 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_manager.cc; 141s ... (16 actions, 4 running)
    [7,999 / 8,118] [Sched] Compiling src/ray/object_manager/pull_manager.cc; 145s ... (16 actions, 5 running)
    [8,001 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_job_manager.cc; 145s ... (16 actions, 4 running)
    [8,001 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_job_manager.cc; 156s ... (16 actions, 4 running)
    [8,001 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_job_manager.cc; 128s ... (16 actions, 5 running)
    [8,002 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_job_manager.cc; 129s ... (16 actions, 4 running)
    [8,002 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/accessor.cc; 128s ... (16 actions, 5 running)
    [8,003 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/accessor.cc; 129s ... (16 actions, 4 running)
    [8,003 / 8,118] [Sched] Compiling src/ray/gcs/gcs_client/accessor.cc; 140s ... (16 actions, 4 running)
    [8,003 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/local_resource_manager.cc; 139s ... (16 actions, 5 running)
    [8,004 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/local_resource_manager.cc; 141s ... (16 actions, 4 running)
    [8,004 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_scheduling_queue.cc; 127s ... (16 actions, 5 running)
    [8,005 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_scheduling_queue.cc; 129s ... (16 actions, 4 running)
    [8,005 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_scheduling_queue.cc; 138s ... (16 actions, 4 running)
    [8,005 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_scheduling_queue.cc; 168s ... (16 actions, 4 running)
    [8,005 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/usage_stats_client.cc; 166s ... (16 actions, 5 running)
    [8,006 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/usage_stats_client.cc; 168s ... (16 actions, 4 running)
    [8,006 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/usage_stats_client.cc; 177s ... (16 actions, 4 running)
    [8,006 / 8,118] [Sched] Compiling src/ray/object_manager/ownership_object_directory.cc; 168s ... (16 actions, 5 running)
    [8,007 / 8,118] [Sched] Compiling src/ray/object_manager/ownership_object_directory.cc; 169s ... (16 actions, 4 running)
    [8,007 / 8,118] [Sched] Compiling src/ray/object_manager/object_manager.cc; 147s ... (16 actions, 5 running)
    [8,008 / 8,118] [Sched] Compiling src/ray/object_manager/object_manager.cc; 148s ... (16 actions, 4 running)
    [8,008 / 8,118] [Sched] Compiling src/ray/object_manager/object_manager.cc; 159s ... (16 actions, 4 running)
    [8,008 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client.cc; 154s ... (16 actions, 5 running)
    [8,009 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client.cc; 156s ... (16 actions, 4 running)
    [8,009 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 133s ... (16 actions, 5 running)
    [8,010 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 134s ... (16 actions, 4 running)
    [8,010 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 145s ... (16 actions, 4 running)
    [8,010 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_task_manager.cc; 148s ... (16 actions, 5 running)
    [8,011 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_task_manager.cc; 150s ... (16 actions, 4 running)
    [8,011 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_task_manager.cc; 155s ... (16 actions, 5 running)
    [8,012 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_task_manager.cc; 156s ... (16 actions, 4 running)
    [8,012 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_worker_manager.cc; 143s ... (16 actions, 5 running)
    [8,013 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_worker_manager.cc; 144s ... (16 actions, 4 running)
    [8,013 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_worker_manager.cc; 155s ... (16 actions, 4 running)
    [8,013 / 8,118] [Sched] Compiling src/ray/object_manager/pull_manager.cc; 157s ... (16 actions, 5 running)
    [8,014 / 8,118] [Sched] Compiling src/ray/object_manager/pull_manager.cc; 159s ... (16 actions, 4 running)
    [8,014 / 8,118] [Sched] Compiling src/ray/object_manager/object_manager.cc; 152s ... (16 actions, 5 running)
    [8,015 / 8,118] [Sched] Compiling src/ray/object_manager/object_manager.cc; 153s ... (16 actions, 4 running)
    [8,015 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_manager.cc; 155s ... (16 actions, 5 running)
    [8,016 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_manager.cc; 156s ... (16 actions, 4 running)
    [8,016 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_worker_manager.cc; 116s ... (16 actions, 5 running)
    [8,017 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_worker_manager.cc; 117s ... (16 actions, 4 running)
    [8,017 / 8,118] [Sched] Compiling src/ray/core_worker/reference_count.cc; 112s ... (16 actions, 5 running)
    [8,018 / 8,118] [Sched] Compiling src/ray/core_worker/reference_count.cc; 113s ... (16 actions, 4 running)
    [8,018 / 8,118] [Sched] Compiling src/ray/core_worker/store_provider/memory_store/memory_store.cc; 116s ... (16 actions, 5 running)
    [8,019 / 8,118] [Sched] Compiling src/ray/core_worker/store_provider/memory_store/memory_store.cc; 118s ... (16 actions, 4 running)
    [8,019 / 8,118] [Sched] Compiling src/ray/core_worker/store_provider/memory_store/memory_store.cc; 127s ... (16 actions, 4 running)
    [8,019 / 8,118] [Sched] Compiling src/ray/core_worker/store_provider/plasma_store_provider.cc; 126s ... (16 actions, 5 running)
    [8,020 / 8,118] [Sched] Compiling src/ray/core_worker/store_provider/plasma_store_provider.cc; 127s ... (16 actions, 4 running)
    [8,020 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 122s ... (16 actions, 5 running)
    [8,021 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 124s ... (16 actions, 4 running)
    [8,021 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/store_client_kv.cc; 134s ... (16 actions, 4 running)
    [8,021 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/bundle_scheduling_policy.cc; 118s ... (16 actions, 5 running)
    [8,022 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/bundle_scheduling_policy.cc; 119s ... (16 actions, 4 running)
    [8,022 / 8,118] [Sched] Compiling src/ray/core_worker/transport/dependency_resolver.cc; 115s ... (16 actions, 5 running)
    [8,023 / 8,118] [Sched] Compiling src/ray/core_worker/transport/dependency_resolver.cc; 116s ... (16 actions, 4 running)
    [8,023 / 8,118] [Sched] Compiling src/ray/core_worker/transport/dependency_resolver.cc; 127s ... (16 actions, 4 running)
    [8,023 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client_pool.cc; 136s ... (16 actions, 5 running)
    [8,024 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client_pool.cc; 137s ... (16 actions, 4 running)
    [8,024 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/composite_scheduling_policy.cc; 120s ... (16 actions, 5 running)
    [8,025 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/composite_scheduling_policy.cc; 122s ... (16 actions, 4 running)
    [8,025 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/bundle_scheduling_policy.cc; 119s ... (16 actions, 5 running)
    [8,026 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/bundle_scheduling_policy.cc; 121s ... (16 actions, 4 running)
    [8,026 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_scheduler.cc; 117s ... (16 actions, 5 running)
    [8,027 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_scheduler.cc; 119s ... (16 actions, 4 running)
    [8,027 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_scheduler.cc; 129s ... (16 actions, 4 running)
    [8,027 / 8,118] [Sched] Compiling src/ray/raylet/worker.cc; 146s ... (16 actions, 5 running)
    [8,028 / 8,118] [Sched] Compiling src/ray/core_worker/future_resolver.cc; 138s ... (16 actions, 5 running)
    [8,029 / 8,118] [Sched] Compiling src/ray/core_worker/future_resolver.cc; 140s ... (16 actions, 4 running)
    [8,029 / 8,118] [Sched] Compiling src/ray/rpc/worker/core_worker_client.cc; 135s ... (16 actions, 5 running)
    [8,030 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 112s ... (16 actions, 5 running)
    [8,031 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 113s ... (16 actions, 4 running)
    [8,031 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 124s ... (16 actions, 4 running)
    [8,031 / 8,118] [Sched] Compiling src/ray/raylet/local_object_manager.cc; 140s ... (16 actions, 5 running)
    [8,032 / 8,118] [Sched] Compiling src/ray/raylet/local_object_manager.cc; 141s ... (16 actions, 4 running)
    [8,032 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/composite_scheduling_policy.cc; 128s ... (16 actions, 5 running)
    [8,033 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/policy/composite_scheduling_policy.cc; 129s ... (16 actions, 4 running)
    [8,033 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 126s ... (16 actions, 5 running)
    [8,034 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 128s ... (16 actions, 4 running)
    [8,034 / 8,118] [Sched] Compiling src/ray/raylet/worker_pool.cc; 137s ... (16 actions, 4 running)
    [8,035 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_task_submitter.cc; 113s ... (16 actions, 4 running)
    [8,035 / 8,118] [Sched] Compiling src/ray/core_worker/transport/actor_task_submitter.cc; 123s ... (16 actions, 4 running)
    [8,035 / 8,118] [Sched] Compiling src/ray/raylet/local_object_manager.cc; 138s ... (16 actions, 5 running)
    [8,036 / 8,118] [Sched] Compiling src/ray/raylet/local_object_manager.cc; 140s ... (16 actions, 4 running)
    [8,036 / 8,118] [Sched] Compiling src/ray/raylet/local_object_manager.cc; 150s ... (16 actions, 4 running)
    [8,037 / 8,118] [Sched] Compiling src/ray/raylet/worker.cc; 143s ... (16 actions, 4 running)
    [8,037 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 141s ... (16 actions, 5 running)
    [8,038 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 142s ... (16 actions, 4 running)
    [8,038 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 153s ... (16 actions, 4 running)
    [8,038 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_retriable_fifo.cc; 126s ... (16 actions, 5 running)
    [8,039 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_retriable_fifo.cc; 127s ... (16 actions, 4 running)
    [8,039 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_retriable_fifo.cc; 137s ... (16 actions, 4 running)
    [8,039 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_scheduler.cc; 150s ... (16 actions, 5 running)
    [8,040 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_resource_scheduler.cc; 152s ... (16 actions, 4 running)
    [8,041 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_server.cc; 148s ... (16 actions, 4 running)
    [8,041 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_task_manager.cc; 149s ... (16 actions, 5 running)
    [8,042 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_task_manager.cc; 151s ... (16 actions, 4 running)
    [8,042 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 125s ... (16 actions, 5 running)
    [8,043 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 127s ... (16 actions, 4 running)
    [8,043 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 136s ... (16 actions, 4 running)
    [8,043 / 8,118] [Sched] Compiling src/ray/raylet/local_task_manager.cc; 150s ... (16 actions, 5 running)
    [8,044 / 8,118] [Sched] Compiling src/ray/raylet/local_task_manager.cc; 152s ... (16 actions, 4 running)
    [8,044 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 151s ... (16 actions, 5 running)
    [8,045 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 152s ... (16 actions, 4 running)
    [8,045 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 162s ... (16 actions, 4 running)
    [8,046 / 8,118] [Sched] Compiling src/ray/raylet/placement_group_resource_manager.cc; 152s ... (16 actions, 4 running)
    [8,046 / 8,118] [Sched] Compiling src/ray/raylet/placement_group_resource_manager.cc; 162s ... (16 actions, 4 running)
    [8,046 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 151s ... (16 actions, 5 running)
    [8,047 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 153s ... (16 actions, 4 running)
    [8,047 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 162s ... (16 actions, 4 running)
    [8,047 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 165s ... (16 actions, 5 running)
    [8,048 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 166s ... (16 actions, 4 running)
    [8,048 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/scheduler_stats.cc; 176s ... (16 actions, 4 running)
    [8,048 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_task_manager.cc; 178s ... (16 actions, 5 running)
    [8,049 / 8,118] [Sched] Compiling src/ray/raylet/scheduling/cluster_task_manager.cc; 180s ... (16 actions, 4 running)
    [8,049 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 164s ... (16 actions, 5 running)
    [8,050 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 166s ... (16 actions, 4 running)
    [8,050 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 176s ... (16 actions, 4 running)
    [8,050 / 8,118] [Sched] Compiling src/ray/core_worker/transport/task_receiver.cc; 175s ... (16 actions, 5 running)
    [8,051 / 8,118] [Sched] Compiling src/ray/core_worker/transport/task_receiver.cc; 177s ... (16 actions, 4 running)
    [8,051 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_group_by_owner.cc; 177s ... (16 actions, 5 running)
    [8,052 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_group_by_owner.cc; 178s ... (16 actions, 4 running)
    [8,053 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_retriable_fifo.cc; 176s ... (16 actions, 4 running)
    [8,054 / 8,118] [Sched] Compiling src/ray/core_worker/actor_manager.cc; 167s ... (16 actions, 4 running)
    [8,054 / 8,118] [Sched] Compiling src/ray/core_worker/actor_manager.cc; 179s ... (16 actions, 4 running)
    [8,054 / 8,118] [Sched] Compiling src/ray/core_worker/actor_manager.cc; 209s ... (16 actions, 4 running)
    [8,054 / 8,118] [Sched] Compiling src/ray/raylet/main.cc; 180s ... (16 actions, 5 running)
    [8,055 / 8,118] [Sched] Compiling src/ray/raylet/main.cc; 182s ... (16 actions, 4 running)
    [8,055 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy_group_by_owner.cc; 181s ... (16 actions, 5 running)
    [8,057 / 8,118] [Sched] Compiling src/ray/core_worker/task_manager.cc; 170s ... (16 actions, 4 running)
    [8,057 / 8,118] [Sched] Compiling src/ray/core_worker/task_manager.cc; 181s ... (16 actions, 4 running)
    [8,057 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 170s ... (16 actions, 5 running)
    [8,058 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 171s ... (16 actions, 4 running)
    [8,058 / 8,118] [Sched] Compiling src/ray/raylet/dependency_manager.cc; 181s ... (16 actions, 4 running)
    [8,058 / 8,118] [Sched] Compiling src/ray/raylet/local_task_manager.cc; 161s ... (16 actions, 5 running)
    [8,059 / 8,118] [Sched] Compiling src/ray/raylet/local_task_manager.cc; 163s ... (16 actions, 4 running)
    [8,059 / 8,118] [Sched] Compiling src/ray/raylet/local_task_manager.cc; 172s ... (16 actions, 4 running)
    [8,059 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 169s ... (16 actions, 5 running)
    [8,060 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 171s ... (16 actions, 4 running)
    [8,060 / 8,118] [Sched] Compiling src/ray/raylet/node_manager.cc; 180s ... (16 actions, 4 running)
    [8,060 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 185s ... (16 actions, 5 running)
    [8,061 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 186s ... (16 actions, 4 running)
    [8,061 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 151s ... (16 actions, 5 running)
    [8,062 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 152s ... (16 actions, 4 running)
    [8,062 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 162s ... (16 actions, 4 running)
    [8,062 / 8,118] [Sched] Compiling src/ray/raylet/placement_group_resource_manager.cc; 166s ... (16 actions, 5 running)
    [8,063 / 8,118] [Sched] Compiling src/ray/raylet/placement_group_resource_manager.cc; 168s ... (16 actions, 4 running)
    [8,063 / 8,118] [Sched] Compiling src/ray/raylet/placement_group_resource_manager.cc; 177s ... (16 actions, 4 running)
    [8,063 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 191s ... (16 actions, 5 running)
    [8,064 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 192s ... (16 actions, 4 running)
    [8,064 / 8,118] [Sched] Compiling src/ray/raylet/raylet.cc; 203s ... (16 actions, 4 running)
    [8,064 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 214s ... (16 actions, 5 running)
    [8,065 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 216s ... (16 actions, 4 running)
    [8,065 / 8,118] [Sched] Compiling src/ray/raylet/worker_killing_policy.cc; 226s ... (16 actions, 4 running)
    [8,065 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_scheduler.cc; 191s ... (16 actions, 5 running)
    [8,066 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_scheduler.cc; 193s ... (16 actions, 4 running)
    [8,066 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_scheduler.cc; 203s ... (16 actions, 4 running)
    [8,066 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_autoscaler_state_manager.cc; 209s ... (16 actions, 5 running)
    [8,067 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_autoscaler_state_manager.cc; 211s ... (16 actions, 4 running)
    [8,067 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_node_manager.cc; 218s ... (16 actions, 5 running)
    [8,068 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_node_manager.cc; 220s ... (16 actions, 4 running)
    [8,068 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 199s ... (16 actions, 5 running)
    [8,069 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 201s ... (16 actions, 4 running)
    [8,069 / 8,118] [Sched] Compiling src/ray/core_worker/transport/normal_task_submitter.cc; 189s ... (16 actions, 5 running)
    [8,070 / 8,118] [Sched] Compiling src/ray/core_worker/transport/normal_task_submitter.cc; 190s ... (16 actions, 4 running)
    [8,070 / 8,118] [Sched] Compiling src/ray/core_worker/transport/normal_task_submitter.cc; 201s ... (16 actions, 4 running)
    [8,070 / 8,118] [Sched] Compiling src/ray/core_worker/transport/normal_task_submitter.cc; 231s ... (16 actions, 4 running)
    [8,070 / 8,118] [Sched] Compiling src/ray/core_worker/object_recovery_manager.cc; 222s ... (16 actions, 5 running)
    [8,071 / 8,118] [Sched] Compiling src/ray/core_worker/object_recovery_manager.cc; 224s ... (16 actions, 4 running)
    [8,071 / 8,118] [Sched] Compiling src/ray/internal/internal.cc; 211s ... (16 actions, 5 running)
    [8,072 / 8,118] [Sched] Compiling src/ray/internal/internal.cc; 213s ... (16 actions, 4 running)
    [8,072 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_scheduler.cc; 212s ... (16 actions, 5 running)
    [8,073 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_scheduler.cc; 214s ... (16 actions, 4 running)
    [8,073 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_scheduler.cc; 224s ... (16 actions, 4 running)
    [8,073 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 218s ... (16 actions, 5 running)
    [8,074 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 219s ... (16 actions, 4 running)
    [8,074 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 230s ... (16 actions, 4 running)
    [8,074 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_resource_manager.cc; 260s ... (16 actions, 4 running)
    [8,074 / 8,118] [Sched] Compiling cpp/src/ray/util/util.cc; 237s ... (16 actions, 5 running)
    [8,075 / 8,118] [Sched] Compiling cpp/src/ray/util/util.cc; 239s ... (16 actions, 4 running)
    [8,076 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_scheduler.cc; 215s ... (16 actions, 4 running)
    [8,076 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_server.cc; 204s ... (16 actions, 5 running)
    [8,078 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_server_main.cc; 184s ... (16 actions, 4 running)
    [8,078 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_server_main.cc; 195s ... (16 actions, 4 running)
    [8,078 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 205s ... (16 actions, 5 running)
    [8,079 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 207s ... (16 actions, 4 running)
    [8,079 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_manager.cc; 216s ... (16 actions, 4 running)
    [8,079 / 8,118] [Sched] Compiling src/ray/core_worker/core_worker.cc; 222s ... (16 actions, 5 running)
    [8,080 / 8,118] [Sched] Compiling src/ray/core_worker/core_worker.cc; 223s ... (16 actions, 4 running)
    [8,080 / 8,118] [Sched] Compiling src/ray/core_worker/core_worker.cc; 234s ... (16 actions, 4 running)
    [8,080 / 8,118] [Sched] Linking raylet; 247s ... (16 actions, 5 running)
    [8,081 / 8,118] [Sched] Linking raylet; 249s ... (16 actions, 4 running)
    [8,081 / 8,118] [Sched] Linking raylet; 258s ... (16 actions, 4 running)
    [8,081 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_scheduler.cc; 208s ... (16 actions, 5 running)
    [8,082 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_actor_scheduler.cc; 209s ... (16 actions, 4 running)
    [8,082 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_autoscaler_state_manager.cc; 202s ... (16 actions, 5 running)
    [8,083 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_autoscaler_state_manager.cc; 203s ... (16 actions, 4 running)
    [8,083 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_autoscaler_state_manager.cc; 214s ... (16 actions, 4 running)
    [8,083 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_node_manager.cc; 218s ... (16 actions, 5 running)
    [8,084 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_node_manager.cc; 219s ... (16 actions, 4 running)
    [8,084 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_node_manager.cc; 230s ... (16 actions, 4 running)
    [8,084 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 220s ... (16 actions, 5 running)
    [8,085 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 221s ... (16 actions, 4 running)
    [8,085 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 231s ... (16 actions, 4 running)
    [8,085 / 8,118] [Sched] Compiling src/ray/gcs/gcs_server/gcs_placement_group_mgr.cc; 261s ... (16 actions, 4 running)
    [8,086 / 8,118] [Sched] Compiling cpp/src/ray/api.cc; 217s ... (16 actions, 4 running)
    [8,086 / 8,118] [Sched] Compiling cpp/src/ray/api.cc; 228s ... (16 actions, 4 running)
    [8,086 / 8,118] [Sched] Compiling cpp/src/ray/config_internal.cc; 232s ... (16 actions, 5 running)
    [8,087 / 8,118] [Sched] Compiling python/ray/_raylet.cpp; 224s ... (16 actions, 5 running)
    [8,088 / 8,118] [Sched] Compiling python/ray/_raylet.cpp; 225s ... (16 actions, 4 running)
    [8,088 / 8,118] [Sched] Compiling python/ray/_raylet.cpp; 235s ... (16 actions, 4 running)
    [8,088 / 8,118] [Sched] Compiling cpp/src/ray/util/process_helper.cc; 241s ... (16 actions, 5 running)
    [8,089 / 8,118] [Sched] Compiling cpp/src/ray/util/process_helper.cc; 243s ... (16 actions, 4 running)
    [8,089 / 8,118] [Sched] Compiling cpp/src/ray/util/process_helper.cc; 253s ... (16 actions, 4 running)
    [8,089 / 8,118] [Sched] Compiling cpp/src/ray/runtime/abstract_ray_runtime.cc; 229s ... (16 actions, 5 running)
    [8,090 / 8,118] [Sched] Compiling cpp/src/ray/runtime/abstract_ray_runtime.cc; 230s ... (16 actions, 4 running)
    [8,090 / 8,118] [Sched] Compiling cpp/src/ray/runtime/abstract_ray_runtime.cc; 240s ... (16 actions, 4 running)
    [8,090 / 8,118] [Sched] Compiling cpp/src/ray/runtime/local_mode_ray_runtime.cc; 235s ... (16 actions, 5 running)
    [8,091 / 8,118] [Sched] Compiling cpp/src/ray/runtime/local_mode_ray_runtime.cc; 236s ... (16 actions, 4 running)
    [8,091 / 8,118] [Sched] Compiling cpp/src/ray/runtime/local_mode_ray_runtime.cc; 246s ... (16 actions, 4 running)
    [8,091 / 8,118] [Sched] Compiling cpp/src/ray/runtime/logging.cc; 217s ... (16 actions, 5 running)
    [8,092 / 8,118] [Sched] Compiling cpp/src/ray/runtime/logging.cc; 218s ... (16 actions, 4 running)
    [8,092 / 8,118] [Sched] Compiling cpp/src/ray/runtime/logging.cc; 228s ... (16 actions, 4 running)
    [8,092 / 8,118] [Sched] Compiling src/ray/core_worker/core_worker_process.cc; 235s ... (16 actions, 5 running)
    [8,093 / 8,118] [Sched] Compiling src/ray/core_worker/core_worker_process.cc; 236s ... (16 actions, 4 running)
    [8,093 / 8,118] [Sched] Executing genrule //:cp_raylet; 236s ... (16 actions, 5 running)
    [8,094 / 8,118] [Sched] Executing genrule //:cp_raylet; 237s ... (16 actions, 4 running)
    [8,094 / 8,118] [Sched] Executing genrule //:cp_raylet; 247s ... (16 actions, 4 running)
    [8,094 / 8,118] [Sched] Compiling cpp/src/ray/runtime/metric/metric.cc; 253s ... (16 actions, 5 running)
    [8,096 / 8,118] [Sched] Compiling cpp/src/ray/runtime/native_ray_runtime.cc; 229s ... (14 actions, 4 running)
    [8,096 / 8,118] [Sched] Compiling cpp/src/ray/runtime/native_ray_runtime.cc; 238s ... (14 actions, 4 running)
    [8,097 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/native_object_store.cc; 179s ... (13 actions, 5 running)
    [8,098 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/native_object_store.cc; 180s ... (12 actions, 4 running)
    [8,098 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/native_object_store.cc; 190s ... (12 actions, 4 running)
    [8,098 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/object_store.cc; 205s ... (12 actions, 5 running)
    [8,099 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/object_store.cc; 207s ... (11 actions, 4 running)
    [8,099 / 8,118] [Sched] Compiling cpp/src/ray/runtime/object/object_store.cc; 216s ... (11 actions, 4 running)
    [8,099 / 8,118] [Sched] Compiling cpp/src/ray/runtime/runtime_env.cc; 201s ... (11 actions, 5 running)
    [8,100 / 8,118] [Sched] Compiling cpp/src/ray/runtime/runtime_env.cc; 202s ... (11 actions, 4 running)
    [8,100 / 8,118] [Sched] Compiling cpp/src/ray/runtime/runtime_env.cc; 212s ... (11 actions, 4 running)
    [8,100 / 8,118] [Sched] Compiling cpp/src/ray/runtime/runtime_env.cc; 242s ... (11 actions, 4 running)
    [8,100 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 233s ... (11 actions, 5 running)
    [8,101 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 234s ... (10 actions, 4 running)
    [8,101 / 8,118] [Sched] Linking src/ray/gcs/gcs_server/gcs_server; 202s ... (10 actions, 5 running)
    [8,102 / 8,118] [Sched] Linking src/ray/gcs/gcs_server/gcs_server; 204s ... (9 actions, 4 running)
    [8,102 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/native_task_submitter.cc; 197s ... (9 actions, 5 running)
    [8,103 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/native_task_submitter.cc; 198s ... (8 actions, 4 running)
    [8,103 / 8,118] [Sched] Compiling cpp/src/ray/util/function_helper.cc; 161s ... (8 actions, 5 running)
    [8,104 / 8,118] [Sched] Compiling cpp/src/ray/util/function_helper.cc; 163s ... (8 actions, 4 running)
    [8,104 / 8,118] [Sched] Compiling cpp/src/ray/util/function_helper.cc; 173s ... (8 actions, 4 running)
    [8,104 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/task_executor.cc; 172s ... (8 actions, 5 running)
    [8,105 / 8,118] [Sched] Compiling cpp/src/ray/runtime/task/task_executor.cc; 174s ... (7 actions, 4 running)
    [8,105 / 8,118] Compiling cpp/src/ray/runtime/object/object_store.cc; 91s linux-sandbox ... (7 actions, 5 running)
    [8,107 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 38s linux-sandbox ... (5 actions, 4 running)
    [8,107 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 41s linux-sandbox ... (5 actions running)
    [8,109 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 43s linux-sandbox ... (4 actions, 3 running)
    [8,111 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 44s linux-sandbox ... (3 actions running)
    [8,111 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 53s linux-sandbox ... (3 actions running)
    [8,111 / 8,118] Compiling cpp/src/ray/runtime/task/local_mode_task_submitter.cc; 83s linux-sandbox ... (3 actions running)
    [8,112 / 8,118] Compiling cpp/src/ray/runtime/task/native_task_submitter.cc; 79s linux-sandbox ... (2 actions running)
    [8,113 / 8,118] Compiling cpp/src/ray/runtime/task/task_executor.cc; 63s linux-sandbox
    [8,113 / 8,118] Compiling cpp/src/ray/runtime/task/task_executor.cc; 73s linux-sandbox
    [8,114 / 8,118] [Prepa] Linking cpp/libray_api.so
    [8,114 / 8,118] Linking cpp/libray_api.so; 1s linux-sandbox
    [8,115 / 8,118] [Prepa] action 'SolibSymlink _solib_k8/_U_S_Scpp_Cray_Ucpp_Ulib___Ucpp/libray_api.so'
    [8,117 / 8,118] Executing genrule //cpp:ray_cpp_pkg; 1s local
    [8,118 / 8,118] checking cached actions
    INFO: Elapsed time: 1993.632s, Critical Path: 851.23s
    INFO: 137 processes: 2 internal, 130 linux-sandbox, 5 local.
    INFO: Build completed successfully, 137 total actions
    # of files copied to build/lib: 415
    Creating /home/owenowenisme/ray/.venv/lib/python3.10/site-packages/ray.egg-link (link to .)
    Adding ray 3.0.0.dev0 to easy-install.pth file
    Installing ray script to /home/owenowenisme/ray/.venv/bin
    Installing serve script to /home/owenowenisme/ray/.venv/bin
    Installing tune script to /home/owenowenisme/ray/.venv/bin

    Installed /home/owenowenisme/ray/python
Successfully installed ray
```



### References
- https://docs.ray.io/en/latest/ray-contribute/development.html
- https://zhuanlan.zhihu.com/p/14305591990