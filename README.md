# msimu
This is a SNMP agent simulation tool/environment for network management system testing, especially simulating the case that massive devices are deployed in the field. Basically, it's based on snmpsim (http://snmplabs.com/snmpsim).

```
#
# Installation & Setup on Ubuntu
#
export http_proxy="http://a.b.c.d:8000"
export https_proxy="http://a.b.c.d:8000"

sudo apt install snmp
sudo apt install libsnmp-dev
sudo apt install virtualenv

mkdir msimu
cd msimu
virtualenv venv
sudo pip install --upgrade pip
pip install snmpsim
source venv/bin/activate

# Put MIB files under $HOME/.snmp/mibs or other valid MIB paths.
# Regarding valid MIB paths, run 'net-snmp-config --default-mibdirs' to check.
# say the MIB files were put under mibs/
mkdir $HOME/.snmp
cp mibs/* $HOME/.snmp/mibs/

snmprec.py --community=nms_snmp --agent-udpv4-endpoint=a.b.c.d --start-oid=1.3.6.1.4.1.7483 --stop-oid=1.3.6.1.5 --output-file=<recfile>.snmprec

# put the captured MIB data as <communicty_str>.snmprec
mv xxx.snmprec ~/.snmpsim/data/<community_str>.snmprec

# install redis
sudo apt install redis

# start one SNMP agent simulator
snmpsimd.py --data-dir=./data --agent-udpv4-endpoint=127.0.0.1:1024
# test the simulator
snmpwalk -v2c -c public 127.0.0.1:1024 system

snmptrap.py -v2c -c nms_snmp 10.243.68.121 123 1.3.6.1.6.3.1.1.5.2 SNMPv2-MIB::sysName.0 = 'mysystem'

# Trap destination register. Add one row in below table
#  - iso.org.dod.internet.snmpV2.snmpModules.snmpTargetMIB.snmpTargetObjects.snmpTargetAddrTable

# FREE mib browser: snmpB (https://sourceforge.net/p/snmpb). Strongly recommend.
```
