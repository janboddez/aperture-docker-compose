# aperture-docker-compose

## Installation
Use something like this (but update the values inside `.env`—and, if needed, `docker-compose.yml`—before kicking off the build step):
```
mkdir ~/www
cd ~/www

git clone https://github.com/janboddez/aperture-docker-compose .

git submodule init
git submodule update

cp .env.example .env

docker-compose build
docker-compose up -d

```
