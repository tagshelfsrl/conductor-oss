# TagShelf Conductor 3.32.4 fork

The `release/3.32.4` baseline is upstream tag `v3.32.4`, commit
`088de237a185caaeb669035a6d75de65d36cab5a`. TagShelf changes are reviewed against
that release branch. Upstream `main` has diverged from this release and is not
the deployment baseline.

The scheduler patch distinguishes an absent task from a failed task lookup,
removes absent references only from the exact polled queue, and observes poll,
load, evaluation, cleanup, capacity, and scheduled-to-first-evaluation delay.
Upstream 3.32.4 inflight reservation, shutdown handling, concurrency limits,
JOIN backoff, persistence, and callback behavior are preserved. See the
[server metrics guide](documentation/metrics/server.md) for units and semantics.

## Owned build and images

`TagShelf Conductor image` checks formatting, runs only the two focused scheduler
test classes with mocked persistence, builds the server with Java 21, and builds
ui-next with Node 22 and its frozen pnpm lockfile. It does not start Conductor,
databases, queues, migrations, or integration tests. The image uses upstream's
`docker/server/Dockerfile.next` with `PREBUILT=true` and includes the existing
PostgreSQL persistence modules and server configuration.

Trusted branch pushes and manual runs publish a Linux amd64 candidate image to
`ghcr.io/tagshelfsrl/conductor-oss:3.32.4-tagshelf-<full commit SHA>` using the
repository's short-lived GitHub token. Pull requests validate without publishing.
The build summary records its immutable image digest. The build sets
`-Pversion=3.32.4` to identify the upstream release; it does not edit the
checked-in component version. Changes to this branch must retain the focused
test selection until broader tests can run within the workspace database policy.

Inherited upstream workflows are retained in `.github/upstream-workflows` as
reference. They are outside GitHub's workflow directory because their publishing
credentials, schedules, and database integration tests do not belong to this fork.
The fork does not use private organization registry credentials.

Before deployment, verify that the candidate package is publicly pullable or
promote the built image to the existing deployment registry. Pin the tested
digest in the staging GitOps overlay. Production continues to use the official
image until the staging behavior and timing results have been reviewed. Building
an image does not deploy it or establish a latency improvement.
