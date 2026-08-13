# Mastodon container cache comparison

This branch builds Mastodon's production `Dockerfile` on the same native
GitHub-hosted runners used by the upstream multi-architecture workflow:
`ubuntu-24.04` for `linux/amd64` and `ubuntu-24.04-arm` for `linux/arm64`.

The workflow keeps three independent cache cohorts:

- GitHub Actions cache uses Mastodon's upstream `type=gha` cache shape and
  `build-${hashFiles('Dockerfile')}-${platform}` scope.
- BoringCache layer cache reads the plan in `layer/.boringcache.toml`.
- BoringCache layer + ccache + mountcache reads the root `.boringcache.toml`.

The separate BoringCache plans are deliberate: their Docker layer tags must
not warm one another. Platform and Git ref scoping remain CLI-owned.

All lanes use the same Dockerfile. It owns the ccache compiler launcher, as
required by the ccache adapter contract; only the composed BoringCache lane
injects remote ccache settings. Mountcache offloads only Mastodon's existing
BuildKit cache mounts for apt plus runtime-verified cache paths for Bundler,
Corepack, and Yarn. The whole `/usr/local/bundle` is restored, reconciled with
`Gemfile.lock`, cleaned, and copied to `/opt/bundle` so the final image retains
it after BuildKit detaches the cache mount. Corepack uses
`/root/.cache/node/corepack`; Yarn uses `/root/.yarn/berry/cache`, matching the
defaults reported by the exact Node 24 and Yarn 4 toolchain used by this image.

The cold seed is based on upstream commit
`37e6c357516cc4c3901edc30b0ce834824750271`. The rolling run merges its exact
child, `b12fe4cda3590fec7bd7aadab5bd64fa65a363df`, which updates FFmpeg from
8.1.2 to 9.0.1 and therefore invalidates a real native compilation layer.
The next rolling run merges upstream commit
`772bf8aee6511b53c96e296eb8622f8ed48596f9`, which updates TypeScript types and
the Yarn lockfile.
