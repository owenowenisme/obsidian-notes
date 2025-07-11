## Commonly used Bazel commands
- bazel query "//src/ray/util/tests:*"
- bazel test :all 
- bazel test //src/ray/gcs/gcs_server/test:gcs_server_rpc_test
- bazel run //:refresh_compile_commands
- bazel build -c fastbuild //:ray_pkg
- bazel clean
