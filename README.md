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
All files and dependencies are fetched the first time the containers are brought up, and configs are automatically set.

Set up and prepare both databases:
```
docker exec -i watchtower_db_1 mysql -u watchtower -psome-random-password watchtower < ./watchtower/html/schema/schema.sql
docker exec watchtower_db_1 mysql -u watchtower -psome-random-password watchtower -e "INSERT INTO users (url, token, created_at) values ('https://aperture.example.org', '1234', NOW());"
```

```
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan migrate
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan key:generate
```

Install the Watchtower service (on the host):
```
# TODO
```

And add the following Watchtower cron job (again, on the host), using `crontab -e`:
```
*/5 * * * * docker exec watchtower_watchtower_1 php /var/www/html/scripts/cron.php > /dev/null 2>&1
```

Once you've set up a token endpoint for your site, you can finally run:
```
docker exec aperture_aperture_1 php aperture/artisan create:user https://example.org/
```
Now all that's left is making sure `https://aperture.example.org`, `https://watchtower.example.org` and `https://camo.example.org` can actually be reached. (Via a web server, and DNS and things.)
