DOWNLOAD THE COD ON :
=>STABLE
LINK:
https://download.moodle.org/releases/latest/

### RUNNING
```sh
pass the file to var/www/html

then Extract the file

create copy config-dist.php and rename it as config.php


setup the CFG:
ex:
$CFG->wwwroot   = 'http://localhost/';

$CFG->dataroot  = '/home/moodledata'; #Create the moodledata folder to store some  uploads...


if some message saying that $CFG->dataroot needed to be configured just run the data-www permission

Run: sudo chown -R www-data:www-data /home/moodledata

check: ls -la /home/moodledata

if it persists

Run: sudo chmod -R 777 /home/moodledata


```
Setting the DB:

$CFG->dbtype    = 'mysqli';      // 'pgsql', 'mariadb', 'mysqli', 'auroramysql', 'sqlsrv' or 'oci'
$CFG->dblibrary = 'native';     // 'native' only at the moment
$CFG->dbhost    = '127.0.0.1';  // eg 'localhost' or 'db.isp.com' or IP
$CFG->dbport    = '3308';  // Database port
$CFG->dbname    = 'moodlebd';     // database name, eg moodle
$CFG->dbuser    = 'root';   // your database username
$CFG->dbpass    = 'password';   // your database password
$CFG->prefix    = 'mdl_';  

```