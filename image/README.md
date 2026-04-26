# Offline Image Archive

The public distribution repo does not store the Docker image tar in git.
GitHub rejects large binary files above the repository file size limit, and the release image can exceed that limit.

Primary install path:

- pull the versioned image from GHCR with `docker compose pull`

Offline path:

- generate an archive from the private source repo with `npm run release:docker:archive`
- or attach the archive separately to a GitHub Release when needed