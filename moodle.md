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
$CFG->wwwroot   = 'http://localhost'; 

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
$CFG->dbname    = 'moodlebd';     // database name, eg moodle
$CFG->dbuser    = 'root';   // your database username
$CFG->dbpass    = 'password';   // your database password
$CFG->prefix    = 'mdl_';  


$CFG->dboptions = array(
    'dbpersist' => false,       // should persistent database connections be
                                //  used? set to 'false' for the most stable
                                //  setting, 'true' can improve performance
                                //  sometimes
    'dbsocket'  => false,       // should connection via UNIX socket be used?
                                //  if you set it to 'true' or custom path
                                //  here set dbhost to 'localhost',
                                //  (please note mysql is always using socket
                                //  if dbhost is 'localhost' - if you need
                                //  local port connection use '127.0.0.1')
    'dbport'    => '3308',          // the TCP port number to use when connecting
                                //  to the server. keep empty string for the
                                //  default port
    'dbhandlesoptions' => false,// On PostgreSQL poolers like pgbouncer don't
                                // support advanced options on connection.
                                // If you set those in the database then
                                // the advanced settings will not be sent.
    'dbcollation' => 'utf8mb4_unicode_ci', /

    ...
);

```
Setting the site:

Developer mode / DEBUG:

//$CFG->debug = (E_ALL | E_STRICT);   // === DEBUG_DEVELOPER - NOT FOR PRODUCTION SERVERS!
// $CFG->debugdisplay = 1;      


On /etc/php/8.1/apache2/php.ini 

put max_input_vars = 5000

then restart apache2
