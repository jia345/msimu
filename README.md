# msimu
This is a simulator for SNMP/NETCONF/etc. in order to test network management system, especially with massive devices deployed. Regarding SNMP part, it's based on snmpsim (http://snmplabs.com/snmpsim).

```
#
# Installation & Setup on Ubuntu
#
sudo apt install snmp
sudo apt install libsnmp-dev
sudo apt install virtualenv

mkdir msimu
cd msimu
virtualenv venv
pip install snmpsim
source venv/bin/activate

# Put MIB files under $HOME/.snmp/mibs or other valid MIB paths.
# Regarding valid MIB paths, run 'net-snmp-config --default-mibdirs' to check.
mkdir $HOME/.snmp
cp -r mib $HOME/.snmp/mibs

# start one SNMP agent simulator
snmpsimd.py --data-dir=./data --agent-udpv4-endpoint=127.0.0.1:1024
# test the simulator
snmpwalk -v2c -c public 127.0.0.1:1024 system
```
