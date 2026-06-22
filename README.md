# go-fsctl docs

mkdocs source for `https://go-fsctl.github.io/docs/`. Built with
[mkdocs-material](https://squidfunk.github.io/mkdocs-material/) and
deployed in versioned form by [mike](https://github.com/jimporter/mike)
via [`.github/workflows/docs.yml`](.github/workflows/docs.yml).

`go-fsctl` is a family of pure-Go (CGO=0) Linux kernel-ioctl control
libraries — [`loop`](https://github.com/go-fsctl/loop),
[`dm`](https://github.com/go-fsctl/dm),
[`btrfs`](https://github.com/go-fsctl/btrfs), and
[`zfs`](https://github.com/go-fsctl/zfs) — with no cgo and no shelling out
to `losetup` / `dmsetup` / `btrfs` / `zpool`.

## Local preview (single-version)

```sh
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve -f mkdocs.yml   # http://127.0.0.1:8000
```

## Deployment

Every push to `main` runs the `docs` workflow, which uses `mike` to
publish the rendered site to the `gh-pages` branch under the version label
`0.1` aliased `latest`. GitHub Pages serves that branch at
`https://go-fsctl.github.io/docs/`, and `/docs/` redirects to the default
(`latest`). The version dropdown in the material theme top bar is wired to
mike via `extra.version.provider: mike` in [`mkdocs.yml`](mkdocs.yml).

## License

BSD-3-Clause. See the `go-fsctl/docs` authors.
