# Deploying paired.exchange

One container serves the built app and the API from the same origin. That is not
an optimization: the frontend only calls `/api/*` relative paths, and the
metadata URLs written on chain must point at the origin that serves them, so a
split deployment introduces a way for those to disagree.

## Live deployment

| | |
|---|---|
| Site | https://paired.exchange |
| Launchpad | `0x6a546350f79DE0Fc83ADfCe99233183aA090fa15` |
| Chain | Robinhood Chain, 4663 |
| Host | Cloud Run, `us-central1`, project `aerial-vehicle-466722-p5` |
| Blob storage | `gs://clink-fun-data`, mounted at `/data` |
| Load balancer | `paired-exchange-*`, static IP `136.68.127.143` |

### Why the cloud resources are still named clink

The project launched as clink.fun. The Cloud Run service (`clink-fun`), its
bucket, service account, attester secret and Artifact Registry repo kept those
names on purpose. The first coins were launched with metadata and image URLs on
`https://clink-fun-93741856042.us-central1.run.app`, and those URLs are
committed on chain forever. Renaming the service would change that hostname and
break every one of them. The run.app URL stays public alongside
`paired.exchange`, and resource names are never shown to users.

## Why Cloud Run and not the alternatives

**GitHub Pages** cannot work. The API is a server: it stores artwork and
descriptors, which cannot live on chain and cannot be produced by a static host.

**Cloudflare Pages** serves the app fine but the API would need rewriting for
Workers, which has no filesystem. That means R2 for blobs and a different storage
adapter. It is a good future move for edge latency and it is not required to
ship.

**Cloud Run** runs the container as written, mounts a bucket where the API
expects a directory, and scales to zero between visitors.

## Storage is not optional

The API writes images and descriptors to `/data`, and the keccak of a
descriptor's exact bytes is committed on chain at launch. If `/data` is
ephemeral, a redeploy breaks every metadata URL ever written, and the on-chain
commitments point at nothing. The GCS volume mount is what makes those URLs
permanent.

## Deploy

```bash
PROJECT=aerial-vehicle-466722-p5
REGION=us-central1
TAG=v8   # one past the tag the service is running now

# Builds the container on Cloud Build and pushes it. The launchpad address is
# compiled into the bundle, so cloudbuild.yaml passes it as a build argument.
gcloud builds submit --config cloudbuild.yaml --region $REGION --project $PROJECT \
  --substitutions=_TAG=$TAG .

gcloud run services update clink-fun --region $REGION --project $PROJECT \
  --image=$REGION-docker.pkg.dev/$PROJECT/clink/web:$TAG
```

The service's env, volume and secret carry over between image updates. To stand
the service up from nothing:

```bash
LAUNCHPAD=0x6a546350f79DE0Fc83ADfCe99233183aA090fa15
gcloud run deploy clink-fun \
  --image=$REGION-docker.pkg.dev/$PROJECT/clink/web:$TAG \
  --project=$PROJECT --region=$REGION \
  --service-account=clink-fun-sa@$PROJECT.iam.gserviceaccount.com \
  --allow-unauthenticated --port=8080 --cpu=1 --memory=512Mi --max-instances=4 \
  --add-volume=name=data,type=cloud-storage,bucket=clink-fun-data \
  --add-volume-mount=volume=data,mount-path=/data \
  --set-env-vars=PAIRED_LAUNCHPAD=$LAUNCHPAD,PAIRED_PUBLIC_URL=https://paired.exchange \
  --set-secrets=PAIRED_ATTESTER_KEY=clink-attester-key:latest
```

`PAIRED_ATTESTER_KEY` signs the origin of every coin planned over MCP. It never
holds funds; losing it only means new MCP launches carry an unsigned origin.
Create it once:

```bash
node -e "process.stdout.write('0x'+require('crypto').randomBytes(32).toString('hex'))" \
  | gcloud secrets create clink-attester-key --data-file=- --project=$PROJECT
gcloud secrets add-iam-policy-binding clink-attester-key --project=$PROJECT \
  --member=serviceAccount:clink-fun-sa@$PROJECT.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

Planned-launch drafts live in `/data/drafts` beside the images and descriptors,
so the same bucket mount keeps them across deploys. The MCP server is served
from the same container at `/mcp`.

`PAIRED_PUBLIC_URL` must be the real public origin. It is baked into every image
and metadata URL the API hands out, and those go on chain.

## First-time setup

```bash
gcloud artifacts repositories create clink --repository-format=docker --location=$REGION
gcloud storage buckets create gs://clink-fun-data --location=$REGION --uniform-bucket-level-access
gcloud iam service-accounts create clink-fun-sa --display-name="paired.exchange runtime"
gcloud storage buckets add-iam-policy-binding gs://clink-fun-data \
  --member=serviceAccount:clink-fun-sa@$PROJECT.iam.gserviceaccount.com \
  --role=roles/storage.objectAdmin
```

## Contracts

```bash
cd contracts
forge install foundry-rs/forge-std --no-git
forge test

PAIRED_TREASURY=0xYourTreasury forge script script/Deploy.s.sol:Deploy \
  --rpc-url https://rpc.mainnet.chain.robinhood.com \
  --private-key $PRIVATE_KEY --broadcast
```

The script deploys the launchpad and registers all 24 stock markets in the same
run, because a launchpad with no enabled quotes is one where every launch reverts.

## Seeding

`scripts/seed.mjs` launches a few real coins through the same path a person uses,
so a fresh deployment is not an empty grid.

```bash
PRIVATE_KEY=0x... node scripts/seed.mjs
```

## Custom domain

`paired.exchange` is served by a global external Application Load Balancer in
front of the Cloud Run service, with a Google-managed certificate. It needs no
domain verification, only A records at the registrar.

| Resource | Name |
|---|---|
| Static IP | `paired-exchange-ip` (`136.68.127.143`) |
| Serverless NEG | `paired-exchange-neg`, pointing at `clink-fun` |
| Backend service | `paired-exchange-backend` |
| Certificate | `paired-exchange-cert`, for `paired.exchange` and `www.paired.exchange` |
| URL map | `paired-exchange-urlmap`: `www` redirects 301 to the apex |
| HTTP redirect | `paired-exchange-http-redirect`: port 80 redirects to HTTPS |
| Forwarding rules | `paired-exchange-https` (443), `paired-exchange-http` (80) |

DNS at the registrar:

| Type | Host | Value |
|---|---|---|
| A | `@` | `136.68.127.143` |
| A | `www` | `136.68.127.143` |

The certificate provisions once both names resolve to the IP, usually within
an hour. Check it with:

```bash
gcloud compute ssl-certificates describe paired-exchange-cert --global \
  --project $PROJECT --format='value(managed.status,managed.domainStatus)'
```

`PAIRED_PUBLIC_URL` points at `https://paired.exchange` only after the
certificate is `ACTIVE`, because the URLs it produces go on chain and must
resolve the moment they are written.
