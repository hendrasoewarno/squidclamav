# squidclamav
Proses instalasi squidclamav

 #siapkan halaman error, misalkan http://localhost/adavirus.html
pico /var/www/html/adavirus.html

<!Doctype html>
<html>
<head>
<title>CDN Safe Browsing</title>
</head>
<body>
<h1>Warning</h1>
<p>
Target URL contain Virus!
</p>
</body>
</html>

#install clamav
apt-get install clamav-daemon

#install c-icap
apt-get install c-icap

#persiapan compile
apt-get install gcc make curl libcurl4-gnutls-dev libicapapi-dev

#compile squidclamav dari source
wget https://github.com/darold/squidclamav/archive/refs/tags/v7.2.tar.gz
tar -xvfz v7.2.tar.gz

./configure --with-c-icap
make
make install

#beralih ke root /
cd

#buat link ke file configurasi
ln -s /etc/c-icap/squidclamav.conf /etc/squidclamav.conf

#setting squidclamav
pico /etc/squidclamav.conf

#beberapa setting yang dipertimbangkan
maxsize ??
redirect http://localhost/adavirus.html
scan_mode ??

#setting c-icap dan tambahkan service squidclamav.so sebagai squidclamav
pico /etc/default/c-icap

set line 6
START=yes

pada /etc/c-icap/c-icap.conf

tambahkan baris
Service squidclamav squidclamav.so

#restart c-icap
systemctl restart c-icap

#autostart clamav-daemonsudo systemctl enable clamav-daemonsudo systemctl start clamav-daemon

#lakukan testing keberhasilan service squidclamav:
c-icap-client -i localhost -p 1344 -s squidclamav

mengembalikan respon
OPTIONS:
Allow 204: Yes
Preview: 1024
Keep alive: Yes

ICAP HEADERS:
ICAP/1.0 200 OK
Methods: RESPMOD, REQMOD
Service: C-ICAP/0.5.6 server - SquidClamav/Antivirus service
ISTag: CI0001-1-squidclamav-10
Transfer-Preview: *
Options-TTL: 3600
Date: Fri, 08 Sep 2023 23:06:27 GMT
Preview: 1024
Allow: 204
X-Include: X-Client-IP, X-Server-IP, X-Authenticated-User, X-Authenticated-Groups
Encapsulated: null-body=

#pastikan c-icap berhasil dijalankan
netstat -na | grep 1344


pico /etc/squid/squid.conf

add lines
icap_enable on
adaptation_send_client_ip on
adaptation_send_username on
icap_client_username_header X-Authenticated-User
icap_client_username_encode off

icap_service service_req reqmod_precache icap://localhost:1344/squidclamav bypass=off
adaptation_access service_avi_req allow all
icap_service service_resp respmod_precache icap://localhost:1344/squidclamav bypass=on
adaptation_access service_avi_resp allow all

systemctl restart squid

test browse to
http://eicar.org/85-0-Download.html

#abaikan semua mime_type yang dimulai dengan audio dan video mime_dont_icapscan.conf

^audio/
^video/

acl domains_dont_icapscan url_regex -i "/etc/squid/domains_dont_icapscan.conf"
acl audio rep_mime_type -i "/etc/squid/mime_dont_icapscan.conf"

icap_service service_req reqmod_precache bypass=1 icap://localhost:1344/ squidclamav
adaptation_access service_req allow !domains_dont_icapscan
icap_service service_resp respmod_precache bypass=1 icap://localhost:1344/ squidclamav
adaptation_access service_resp allow !domains_dont_icapscan !audio
