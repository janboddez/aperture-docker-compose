# Aperture (docker-compose)
Run Aperture—an IndieWeb feed aggregator—using Docker and docker-compose.

## Installation
Use something like this:
```
mkdir ~/www
cd ~/www

git clone https://github.com/janboddez/aperture-docker-compose .

git submodule init
git submodule update

cp .env.example .env
```
Then, before kicking off the build step, update the values inside `.env`—and, if needed, `docker-compose.yml`.

This little README assumes your main site lives at `https://example.org`, and that Aperture will be set up at `https://aperture.example.org` and Watchtower at `https://watchtower.example.org`. You'll want to replace those three URLs with your own.

Make sure, when editing `.env`, to fill out an actual Redis password (the way `docker-compose.yml` is set up, `null` won't do). Or modify `docker-compose.yml` according to your preferred setup.

If you're going to go with the prebuilt images in Docker's repository, comment out the `build` entries for `aperture` and `watchtower` in `docker-compose.yml`. (This'll save you some time.)

Launch the different containers:
```
docker-compose build
docker-compose up -d
```
All files and dependencies are fetched the first time the containers are brought up, and configs are automatically set. This takes a while, so, er, maybe wait a minute or two before moving on?

Now, set up and run migrations for both databases. First up is Watchtower's. (Replace `1234` with your token of choice—see `.env`. The same goes your Watchtower database password.)
```
docker exec -i watchtower_db_1 mysql -u watchtower -psome-random-password watchtower < ./watchtower/html/schema/schema.sql
docker exec watchtower_db_1 mysql -u watchtower -psome-random-password watchtower -e "INSERT INTO users (url, token, created_at) values ('https://aperture.example.org', '1234', NOW());"
```
(I know, you _shouldn't_ just type passwords into a terminal like that, but doing so allows us to set everything up with a single command. Feel free to run `history -c` afterward to make your system forget.)

And then Aperture's.
```
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan migrate
docker exec aperture_aperture_1 php /var/www/html/aperture/artisan key:generate
```

Install the Watchtower service (on the host). This'll make sure Watchtower (inside its container) keeps doing its thing:
```
sudo cp ~/www/watchtower/build/watchtower.service /etc/systemd/watchtower.service
sudo systemctl enable watchtower
sudo systemctl start watchtower
```

This last step, right now, takes 15 seconds because of a pre-start delay that may or may not make sense. (Anyway, _don't just terminate it_ if nothing seems to happen.)

And add the following Watchtower cron job (again, on the host), using `crontab -e`:
```
*/5 * * * * docker exec watchtower_watchtower_1 php /var/www/html/scripts/cron.php > /dev/null 2>&1
```

To make sure Aperture and its dependencies automatically start after a reboot, you may also want to add:
```
@reboot cd ~/www && docker-compose up -d
```

Almost there! Once you've set up a token endpoint for your site, run:
```
docker exec aperture_aperture_1 php aperture/artisan create:user https://example.org/
```

Now all that's left is making sure `https://aperture.example.org` and `https://watchtower.example.org` can actually be reached. (Via a web server, and DNS and things.)

Finally, refer to Aperture from your main site, like so:
```
<link rel="microsub" href="https://aperture.example.org/microsub/1">
```
