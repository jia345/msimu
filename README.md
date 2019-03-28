# msimu
This is a SNMP agent simulation tool/environment for network management system testing, especially simulating the case that massive devices are deployed in the field. Basically, it's based on snmpsim (http://snmplabs.com/snmpsim).

```
#
# Installation & Setup on Ubuntu
#
export http_proxy="http://a.b.c.d:8000"
export https_proxy="http://a.b.c.d:8000"

# sudo apt install snmp  # optional
# sudo apt install libsnmp-dev  # optional
sudo apt install virtualenv
sudo apt install redis
sudo pip install --upgrade pip

mkdir msimu
cd msimu
virtualenv venv
source venv/bin/activate
pip install snmpsim
pip install snmpclitools
pip install redis

sudo service mariadb start   # /bin/systemctl start mariadb.service
mysql -u root -p
MariaDB [(none)]> set GLOBAL max_connections=2000;
MariaDB [(none)]> FLUSH HOSTS;

#
# PERFORMANCE IMPROVEMENT
#
vim snmpsim/variation/sql.py # change all %10s to %5s to reduce the length of oid

# Put MIB files under $HOME/.snmp/mibs or other valid MIB paths.
# Regarding valid MIB paths, run 'net-snmp-config --default-mibdirs' to check.
# say the MIB files were put under mibs/
mkdir $HOME/.snmp
cp mibs/* $HOME/.snmp/mibs/

# snmprec.py --community=nms_snmp --agent-udpv4-endpoint=a.b.c.d --start-oid=1.3.6.1.4.1.7483 \
# --stop-oid=1.3.6.1.5 --output-file=<recfile>.snmprec --variation-module=redis \
# --variation-module-options=host:127.0.0.1,port:6379,db:0,key-spaces-id:1830
#snmprec.py --community=nms_snmp --agent-udpv4-endpoint=135.251.97.134 --output-file=recdata/pss8-sql.snmprec \
#           --variation-module=sql --variation-module-options=dbtype:sqlite3,database:recdata/sqlite.db,dbtable:snmprec
#snmprec.py --agent-udpv4-endpoint=135.251.97.133 --use-getbulk --community=nms_snmp --output-file=recdata/mysql-ne133.snmprec \
#           --variation-module=sql --variation-module-options=dbtype:mysql.connector,user:root,database:snmpsim
snmprec.py --agent-udpv4-endpoint=135.251.97.134 --use-getbulk --community=nms_snmp --output-file=recdata/mysql-ne134-oid5.snmprec --variation-module=sql --variation-module-options=dbtype:mysql.connector,user:root,database:snmpsim,dbtable:snmprec_oid5

cp recdata/pss8-sql.snmprec ~/.snmp/data/nms_snmp.snmprec

# put the captured MIB data as <communicty_str>.snmprec
mv xxx.snmprec ~/.snmpsim/data/<community_str>.snmprec

# install redis
sudo apt install redis

MariaDB [none]> show global variables like '%timeout%';
MariaDB [none]> set global wait_timeout=28800;
MariaDB [none]> set global interactive_timeout=28800;

# indexing captured SQL data
MariaDB [none]> use snmpsim;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [snmpsim]> show tables;
+-------------------+
| Tables_in_snmpsim |
+-------------------+
| pss8_ne134_rec    |
| snmprec           |
| snmprec_oid5      |
+-------------------+
3 rows in set (0.00 sec)
MariaDB [snmpsim]> describe pss8_ne134_rec;
+-----------+------+------+-----+---------+-------+
| Field     | Type | Null | Key | Default | Extra |
+-----------+------+------+-----+---------+-------+
| oid       | text | YES  |     | NULL    |       |
| tag       | text | YES  |     | NULL    |       |
| value     | text | YES  |     | NULL    |       |
| maxaccess | text | YES  |     | NULL    |       |
+-----------+------+------+-----+---------+-------+
4 rows in set (0.00 sec)
MariaDB [snmpsim]> select count(*) from pss8_ne134_rec;
+----------+
| count(*) |
+----------+
|   232420 |
+----------+
1 row in set (0.20 sec)
MariaDB [snmpsim]> select oid from pss8_ne134_rec where oid>'    1.    3.    6.    1.    4.    1. 7483.    2.    1.    1.    2.    1.    1.    0'  limit 1;
Empty set (0.22 sec)

MariaDB [snmpsim]> create unique index oidIndex on pss8_ne134_rec(oid(767));
Query OK, 0 rows affected (3.18 sec)
Records: 0  Duplicates: 0  Warnings: 0

MariaDB [snmpsim]> select oid from pss8_ne134_rec where oid>'    1.    3.    6.    1.    4.    1. 7483.    2.    1.    1.    2.    1.    1.    0'  limit 1;
Empty set (0.00 sec)


# edit venv/snmpsim/variation/sql.py to remove 'order by oid'
<<<<
cursor.execute('select oid from %s where oid>\'%s\' order by oid limit 1' % (dbTable, sqlOid))
------
cursor.execute('select oid from %s where oid>\'%s\' limit 1' % (dbTable, sqlOid))
>>>>
# to add a ping thread in init()
17a18,19
> import threading, time, signal
> import sys
25a28,32
> def pingFunc(dbConn):
>     while True:
>         time.sleep(15)
>         #cursor = dbConn.cursor()
>         dbConn.ping(True)
78a86,90
>     else: # start a ping thread
>         pass
>         ping = threading.Thread(target = pingFunc, args = (dbConn,))
>         ping.setDaemon(True)
>         ping.start()


# start one SNMP agent simulator
# snmpsimd.py --data-dir=./data --agent-udpv4-endpoint=0.0.0.0:30161
# snmpsimd.py --variation-module-options=redis:host:127.0.0.1,port:6379,db:0 --agent-udpv4-endpoint=0.0.0.0:30161
# snmpsimd.py --variation-module-options=sql:dbtype:sqlite3,database:recdata/sqlite.db --agent-udpv4-endpoint=0.0.0.0:30161
snmpsimd.py --variation-module-options=sql:dbtype:mysql.connector,host:127.0.0.1,port:3306,user:root,database:snmpsim --agent-udpv4-endpoint=0.0.0.0:30161
# test the simulator
snmpwalk -v2c -c public 127.0.0.1:1024 system
snmptrap.py -v2c -c nms_snmp 10.243.68.121 123 1.3.6.1.6.3.1.1.5.2 SNMPv2-MIB::sysName.0 = 'mysystem'

# Trap destination register. Add one row in below table
#  - iso.org.dod.internet.snmpV2.snmpModules.snmpTargetMIB.snmpTargetObjects.snmpTargetAddrTable
sudo apt install npm
sudo npm install vue-cli -g
python3 -m venv venv3
source vent3/bin/activate
pip install Flask

git clone https://github.com/iview/iview-project.git bignotify
cd bignotify
npm config set registry https://registry.npm.taobao.org
npm install
npm run init
npm run dev
npm run build // for production

#
# wireshark to capture SNMP packets
#
sudo apt-get install wireshark
sudo usermod -a -G wireshark $USER
tshark -Y "snmp && (ip.src == 135.251.97.134 || ip.dst == 135.251.97.134)" -i ens192

# FREE mib browser: snmpB (https://sourceforge.net/p/snmpb). Strongly recommend.
```
#
# CentOS
#
pip install mysql-connector-python
