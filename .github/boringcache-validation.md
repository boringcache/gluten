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

# BoringCache Velox cross-workflow validation

## Upstream problem

[Apache Gluten issue #12743](https://github.com/apache/gluten/issues/12743)
reports that `velox_backend_x86.yml` and `delta_spark_ut.yml` independently
build the same CentOS 7 Velox native library. Changes under `gluten-delta/**`
and `backends-velox/src-delta*/**` can start both workflows even though those
paths do not change the native-library inputs. GitHub workflow artifacts cannot
be shared between the two independent workflow runs.

The validation therefore modifies those two workflows directly. The previous
sequential publish/restore demonstration was removed because it did not
exercise the reported workflow boundary.

## Upstream reference

The successful upstream Delta workflow
[run 34931227945](https://github.com/apache/gluten/actions/runs/34931227945)
used source `9f6dcb599d3ebb8ef9f55450163aae4f025300e9`. Its native-build job took
3m26s, including 2m30s in `Build Gluten native libraries`. The complete
workflow took 2h04m27s of wall time and approximately 668.1 runner-minutes.

The successful upstream Velox x86
[run 34427915258](https://github.com/apache/gluten/actions/runs/34427915258)
used an earlier source revision. Its native-build job took 5m30s, including
4m37s in `Build Gluten native libraries`. The complete workflow took 1h49m30s
of wall time and approximately 1,581.5 runner-minutes. This is an upstream
reference, not a same-revision provider comparison.

## Validation design

Both real native-build jobs now:

1. Calculate and verify the same committed input identity.
2. Restore `cpp/build` from BoringCache through GitHub OIDC.
3. Accept the restore only when the embedded full input digest matches and
   `cpp/build/releases` contains output.
4. Restore the existing Apache Stash ccache and run Gluten's unchanged native
   build on a miss or invalid restore.
5. Embed the input digest in a successful build before BoringCache publishes
   it.
6. Upload the existing workflow artifact so every downstream job keeps its
   original contract.

The input digest covers the tracked `cpp`, `dev`, and `ep/build-velox` trees;
the exact Velox commit; the immutable native-builder image digest;
`NUM_THREADS=4`; and the native build command. The BoringCache archive tag uses
the full digest and disables Git-derived suffixes so both workflows resolve the
same immutable entry.

Pull-request and comment-triggered jobs are restore-only. Trusted dispatch and
scheduled jobs may publish. Cache misses and cache transport errors fall back
to the existing native build. Apache Stash remains the compiler-cache provider
for that fallback. No static BoringCache credential is stored in the
repository.

## Results

Full-workflow BoringCache results are pending. Do not use the earlier
sequential validation run as evidence that the two upstream workflows avoid
their duplicate native build.
