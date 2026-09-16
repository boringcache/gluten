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

## Issue-bounded result

| Upstream pain | Exact experiment | Measured result | Bounded verdict |
| --- | --- | --- | --- |
| [Issue #12743](https://github.com/apache/gluten/issues/12743) reports that the independent Velox x86 and Delta workflows build the same CentOS 7 native library because workflow artifacts cannot cross the workflow-run boundary. | Run the complete x86 and Delta workflows with one content-addressed native archive: one workflow builds and publishes, and the sibling restores it. Repeat after a real native-input change to verify a miss, rebuild, and reverse-direction reuse. | At the exact upstream source, the two native jobs fell from 7m02s upstream to 4m41s with BoringCache, a 2m21s or 33.4% reduction. The Delta consumer restored 685.22 MB in 4.9s and took 52s. After the native-input change, Delta rebuilt and published; x86 restored 685.21 MB in 5.9s and took 36s. | This addresses the issue's duplicate native-build boundary. It does not establish that BoringCache makes the complete x86 or Delta workflow broadly faster; unrelated test variance, a Maven Central 404, and workflow concurrency cancellation affected the whole-workflow outcomes. |

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

### Same-source cross-workflow reuse

Velox x86
[run 35021209096](https://github.com/boringcache/gluten/actions/runs/35021209096)
bootstrapped the exact native-library entry at workflow commit
`b02de0e0913f664945af42bb9387de48a2a868a3`. Its native job took 3m49s,
including a 2m52s build with the existing Apache Stash ccache and a 5s
BoringCache post step. The native job passed. The complete x86 dispatch ended
`cancelled` after 6h04m27s when five Spark jobs reached GitHub's six-hour job
limit; the other 47 jobs passed. It used approximately 3,385.4 runner-minutes.

Delta
[run 35023227002](https://github.com/boringcache/gluten/actions/runs/35023227002)
then used the same source and input digest in the independent
`delta_spark_ut.yml` workflow. BoringCache restored 685.22 MB and 2,400 files
in 4.9s: 186ms for the archive graph, 3.4s for 111 blobs, and 1.3s to
materialize the files. The complete BoringCache Action step, including CLI and
OIDC setup, took 12s. Digest validation passed, and the workflow skipped both
Apache Stash and the native build.

The Delta native job took 52s, compared with 3m26s in exact-source upstream
[run 34931227945](https://github.com/apache/gluten/actions/runs/34931227945),
a 2m34s or 74.8% reduction. The BoringCache Delta workflow then passed its real
bundle build, all eight test shards, and aggregation. It used 654.5
runner-minutes and finished in 1h46m51s. Upstream used 668.1 runner-minutes and
finished in 2h04m27s. The direct cache result is the 2m34s native-job reduction;
bundle and test-shard variance also affects the 13.6 runner-minute and 17m36s
whole-workflow differences.

On the exact source, upstream x86
[run 34919759644](https://github.com/apache/gluten/actions/runs/34919759644)
spent 3m36s in its native job, including 2m43s in the native build, before five
unrelated matrix jobs later reached the six-hour cancellation limit. That
upstream run passed the same other 47 jobs, ended after 6h04m15s, and used
approximately 3,089.3 runner-minutes. The additional BoringCache-fork runner
time came from variance in unrelated jobs, including separate UDF and cuDF
native builds; it is not evidence about the shared CentOS 7 archive. Across the
two upstream native jobs, x86 and Delta used 7m02s. The BoringCache pair used
4m41s because only x86 built the native library, a 2m21s or 33.4% reduction
across the reported duplicate-build boundary.

### Rolling native-input change

Upstream commit `f04968b1083b12c0a13581fdaa36003b3fd1a87f` changes `cpp/**`
and therefore produces a different full input digest. Delta
[run 35033798917](https://github.com/boringcache/gluten/actions/runs/35033798917)
correctly missed the earlier entry, restored Apache Stash, built the native
library in 2m52s, embedded the new digest, and published the new entry. Its
native job took 4m00s. The complete workflow passed its bundle build, all
eight test shards, and aggregation. It used approximately 630.0 runner-minutes
and finished in 2h06m32s. The whole-workflow difference from the first Delta
run reflects bundle and test-shard variance and is not a cache-performance
comparison.

The independent x86
[run 35043780706](https://github.com/boringcache/gluten/actions/runs/35043780706)
then restored that rolling-source entry. Its native job took 36s. The internal
restore took 5.9s for 685.21 MB and 2,400 files: 303ms for the archive graph,
4.2s for 103 blobs, and 1.3s to materialize the files. Digest validation passed,
the job skipped Apache Stash and native compilation, and the existing artifact
upload passed.

The rolling x86 dispatch ended `cancelled` after 1h04m52s with approximately
1,566.1 runner-minutes. It had passed 44 jobs. One downstream TPC lane failed
after Maven Central returned HTTP 404 for
`apache-maven-3.9.16-bin.tar.gz`; the equivalent lane in the first x86 run had
downloaded that exact file successfully five hours earlier. A delayed scheduled
[run 35047979375](https://github.com/boringcache/gluten/actions/runs/35047979375)
then started on the same commit and concurrency key, cancelled the remaining
seven jobs, and was itself skipped. These failures happened after the native
job had validated and uploaded the restored artifact.

Across both source identities, the real workflow sequence was:

| Native input | Publisher | Consumer | Combined native-job time |
| --- | --- | --- | ---: |
| `9f6dcb599` | x86: miss and build in 3m49s | Delta: restore in 52s | 4m41s |
| `f04968b10` | Delta: miss and build in 4m00s | x86: restore in 36s | 4m36s |

Each source change produced a distinct entry, each miss kept Apache Stash and
the normal build fallback, and each sibling workflow consumed only the entry
whose embedded full digest matched its inputs. Apache did not run either target
workflow on `f04968b10`, so there is no same-revision upstream performance
baseline for the rolling pair.
