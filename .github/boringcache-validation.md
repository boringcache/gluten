<!--
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# BoringCache Velox native-library validation

## Result

[Run 34988432841](https://github.com/boringcache/gluten/actions/runs/34988432841)
published the Velox native build output and restored it in a later job on a
clean GitHub-hosted runner. The run used Gluten source
`9f6dcb599d3ebb8ef9f55450163aae4f025300e9`, validation workflow commit
`c180e8fc684fbb46b7bd5bd4ce52a5d117a5371f`, and BoringCache action v1.30.4
at immutable commit `1039999c65011be670f5655e0e48ad556188ab12`.

| Measurement | Publish job | Clean-runner restore job |
| --- | ---: | ---: |
| Complete job | 3m12s | 19s |
| Native build from an empty `cpp/build` | 2m33.22s | Not run |
| BoringCache archive operation | 6.5s save | 3.6s restore |
| Archive contents | 685.22 MB, 2,400 files | 685.22 MB, 2,400 files |
| Restored `cpp/build` disk use | Not measured | 658 MB |
| Restored-output validation | Not applicable | Less than 0.1s |

The archive save created manifest
`sha256:329ed4ff81098b495970aa0b340ae5dbaf674d9d9e4a97207bd77e1a54ac9ddc`.
The clean-runner job restored the same manifest. Its resolver reported 153.49 MB
before downloading 111 archive blobs in 2.1 seconds and materializing the
archive in 1.3 seconds. The complete 19-second job also included checkout,
OIDC connection, action setup, validation, and evidence upload.

The publish job removed `cpp/build` before invoking Gluten's native build. Its
Apache Stash lookup restored the compiler cache saved by the earlier setup run,
so ccache reported 1,044 hits and no misses. The 2m33.22s build time therefore
measures clean native output with a warm compiler cache. It is not a cold-build
baseline.

## Correctness boundary

The archive tag is derived from an input digest covering the `cpp`, `dev`, and
`ep/build-velox` Git trees; the pinned builder image digest; the exact Velox
commit; and the build flags. The publish job writes this digest into the native
output. The restore job compares that embedded digest with the expected value
and checks that `cpp/build/releases` contains a file before accepting the
restore.

The publish job is restricted to this fork's `boringcache-validation` branch.
The restore job has restore-only access. Both jobs connect through GitHub OIDC;
the repository does not store a BoringCache token. The validation preserves
Gluten's Apache Stash ccache path instead of replacing it.

## Public reference and limitations

Apache Gluten's
[PR #12820 job](https://github.com/apache/gluten/actions/runs/32232899547/job/96006408928)
reported 9m25s for the native build step and 10m39s for the complete job. The
BoringCache validation restored the matching native output in 3.6 seconds and
completed its clean-runner restore job in 19 seconds, but this is not a
controlled provider comparison. The Apache job used a different source
revision and its existing Apache Stash state.

[Run 34979095023](https://github.com/boringcache/gluten/actions/runs/34979095023)
was the initial setup attempt. Its Apache Stash lookup missed, its compiler-cold
native build took 1h19m59s, and a host-side permission error prevented the
BoringCache archive from being published. The subsequent workflow correction
writes the validation digest inside the existing build container. The failed
run is setup evidence, not performance evidence.

The successful publish run's 100% ccache hit rate also means its 2m33.22s build
must not be compared directly with Apache's 9m25s native build. The supported
result is narrower: for this content-bound native output, BoringCache saved the
archive in 6.5 seconds and restored it on a clean runner in 3.6 seconds. This
validation does not run the downstream Maven bundle or test workflows.

