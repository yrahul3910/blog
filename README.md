## Setup

Install [Hugo](https://gohugo.io).

```sh
git submodule update --init --recursive
```

## Running locally

```sh
hugo server --buildDrafts
```

## Deployment

Make sure you've authenticated with Google Cloud using:

```sh
gcloud auth application-default login
```

Run the following:

```sh
hugo
hugo deploy --target=production
```
