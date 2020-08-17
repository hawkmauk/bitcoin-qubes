# bitcoin-qubes
Installing and configuring bitcoin on qubes-os based on the [qubenix guide](https://github.com/qubenix/qubes-whonix-bitcoin/blob/master/0_bitcoind.md)

# Creating TemplateVM
```[sh]
[user@dom0] qvm-clone whonix-ws-15 whonix-ws-15-bitcoin
```

# Install bitcoin
Open the TemplateVM and create a bitcoin user
```[sh]
[user@host] groupadd bitcoin
[user@host] useradd -G bitcoin -H -u bitcoin bash
[bitcoin@host] cd
```

clone the bitcoin repository
```[sh]
[bitcoin@host] git clone git@github.com:bitcoin/bitcoin.git
[bitcoin@host] cd bitcoin
```
Install BerkleyDB
```[sh]
[bitcoin@host: ~/bitcoin] ./contrib/install_db4.sh `pwd`
[bitcoin@host: ~/bitcoin] export BDB_PREFIX='/home/bitcoin/bitcoin/db4'
```

Build bitcoin
This builds bitcoin and installs the binaries in the directory /home/bitcoin/bitcoin/bin/
```[sh]
[bitcoin@host: ~/bitcoin] ./autogen.sh
[bitcoin@host: ~/bitcoin] ./configure BDB_LIBS="-L${BDB_PREFIX}/lib -ldb_cxx-4.8" \
BDB_CFLAGS="-I${BDB_PREFIX}/include" --prefix=/home/bitcoin
[bitcoin@host: ~/bitcoin] make check && make install  
```

Copy files to shared locations as the local bitcoin user directory will not be available to the AppVM
```[sh]
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/bin/* /usr/bin/
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/lib/* /usr/lib/
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/lib/pkgconfig/* /usr/lib/pkgconfig/
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/include/* /usr/include/
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/share/* /usr/share/bitcoin
```

## Configure bitcoind.service
First we copy the service configuration to the systemd lib folder.
We then add a hook into the qube configuration to access it from the services tab
```[sh]
[user@whonix-ws-15-bitcoin] sudo cp /home/bitcoin/bitcoin/contrib/init/bitcoind.service \
/lib/systemd/system/
[user@whonix-ws-15-bitcoin] sudo mkdir /etc/systemd/system/bitcoind.service.d/
[user@whonix-ws-15-bitcoin] sudo cat << EOF >> /etc/systemd/system/bitcoind.service.d/30_qubes.conf
[Unit]
ConditionPathExists=/var/run/qubes-service/bitcoind
EOF
```

### bitcoind.service
```
[Unit]
Description=Bitcoin daemon
After=qubes-sysinit.target

[Service]
ExecStart=/usr/bin/bitcoind -daemon \
                            -pid=/run/bitcoind/bitcoind.pid \
                            -conf=/home/bitcoin/.bitcoin/bitcoin.conf \
                            -datadir=/home/bitcoin/.bitcoin
ExecStop=/usr/bin/bitcoin-cli stop

# Process management
####################

Type=forking
PIDFile=/run/bitcoind/bitcoind.pid
Restart=on-failure
TimeoutStopSec=600

# Directory creation and permissions
####################################

# Run as bitcoin:bitcoin
User=bitcoin
Group=bitcoin

# /run/bitcoind
RuntimeDirectory=bitcoind
RuntimeDirectoryMode=0710

# Hardening measures
####################

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Deny access to /home, /root and /run/user
# - need to allow this to save data in the bitcoin users home directory
ProtectHome=false

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Use a new /dev namespace only populated with API pseudo devices
# such as /dev/null, /dev/zero and /dev/random.
PrivateDevices=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

[Install]
WantedBy=multi-user.target
```

# Creating AppVM
Create the AppVM
extend the private diak allocation to 350GB
Enable the bitcoind service
```[sh]
[user@dom0 ~] qvm-create --label green --prop netvm='sys-bitcoin' \
--template whonix-ws-15-bitcoin bitcoin-mainnet
[user@dom0 ~] qvm-volume extend bitcoin-mainnet:private 350GB
[user@dom0] qvm-service -e bitcoin-mainnet bitcoind
```
