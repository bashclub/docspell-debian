Docspell Installation Debian mit allem

Wir starten mit priviligiertem Container

```
apt update && apt full-upgrade
apt install curl htop zip gnupg2 ca-certificates sudo
apt install default-jdk apt-transport-https wget -y
apt install ghostscript tesseract-ocr tesseract-ocr-deu tesseract-ocr-eng unpaper unoconv wkhtmltopdf ocrmypdf
```

```
vi /etc/systemd/system/unoconv.service


[Unit]
Description=Unoconv listener for document conversions
Documentation=https://github.com/dagwieers/unoconv
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=simple
Environment="UNO_PATH=/usr/lib/libreoffice/program"
ExecStart=/usr/bin/unoconv --listener

[Install]
WantedBy=multi-user.target


systemctl enable unoconv.service
```

Download Solr

```
cd
curl https://nc.cloudistboese.de/index.php/s/teWBKk4xBeo6bXA/download > solr-8.11.1.tgz
tar xzf solr-8.11.1.tgz
bash solr-8.11.1/bin/install_solr_service.sh solr-8.11.1.tgz

systemctl start solr

su solr -c '/opt/solr-8.11.1/bin/solr create -c docspell'
```

Postgres Install

```
curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/apt.postgresql.org.gpg >/dev/null
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ bullseye-pgdg main" > /etc/apt/sources.list.d/postgresql.list'
apt update && apt full-upgrade
apt install postgresql-14
```

```
sudo su -
su - postgres
psql


CREATE USER docspell
WITH SUPERUSER CREATEDB CREATEROLE
PASSWORD 'DocSpell123';

CREATE DATABASE docspelldb WITH OWNER docspell;
\connect docspelldb;
\q

exit

systemctl enable postgresql
```

```
cd /tmp
wget https://github.com/eikek/docspell/releases/download/v0.36.0/docspell-joex_0.36.0_all.deb
wget https://github.com/eikek/docspell/releases/download/v0.36.0/docspell-restserver_0.36.0_all.deb
dpkg -i docspell*

service docspell-restserver stop
service docspell-joex stop


mkdir /home/docspell
mkdir /home/docspell/.cache
touch /home/docspell/.cache/dconf
chown -R docspell:docspell /home/docspell/
```

```
cd
wget https://github.com/docspell/dsc/releases/download/v0.9.0/dsc_amd64-musl-0.9.0
mv dsc_amd* dsc
chmod +x dsc
mv dsc /usr/bin
```

```
In /etc/docspell-joex/docspell-joex.conf und
   /etc/docspell-restserver/docspell-server.conf
folgende Änderungen machen:

  full-text-search {
    # The full-text search feature can be disabled. It requires an
    # additional index server which needs additional memory and disk
    # space. It can be enabled later any time.
    #
    # Currently the SOLR search platform and PostgreSQL is supported.
    enabled = true

sowie

jdbc {
   url = "jdbc:postgresql://localhost:5432/docspelldb"
   user = "docspell"
   password = "DocSpell123"
 }
Dieser Eintrag kommt in jeder Konfiguration 2 mal vor, einmal unter Frontend und einmal unter Backend. Es müssen beide Einträge editiert werden!

vi /etc/docspell-joex/docspell-joex.conf 

    pool-size = 8

#Paralelle Jobs
```

```
apt install nginx
openssl dhparam -out /etc/nginx/dhparam.pem 2048


mkdir /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/docs.home.key -out /etc/nginx/ssl/docs.home.crt

curl https://raw.githubusercontent.com/andreklug/docspell-debian/main/nginx-default > /etc/nginx/sites-enabled/default 



auskommentieren oder löschen
vi /etc/nginx/sites-enabled/default

/etc/nginx/ssl/homelab.local_CA.crt

service nginx restart
```

```
root@Docspell:/mnt/import# cat import.sh 
dsc login -u docspell --password DocSpell123
find -type f -iname '*.pdf' -exec dsc upload "{}" \;
find -type f -iname '*.doc' -exec dsc upload "{}" \;
```
