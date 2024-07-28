# CVMFS on MacBooks

Instructions to run cvmfs and grid scripts on macbook

Assumes you have `xcode` command line tools (see [xcode](https://developer.apple.com/xcode/resources/)) installed on your macbook and preferably use `iterm` as your terminal and `zsh` as your default shell.

## Downloading requirements

If you want to work on `cvmfs` on macbooks with full fledged docker containers, do the following:
- Download & install Docker Desktop daemon from [Docker-Official](https://www.docker.com/products/docker-desktop/)
- Download & install macFUSE from [macFUSE](https://osxfuse.github.io)
   - This will require you to allow the package by Benjamin F from Settings> Privacy & Security>Privacy
   - Click on allow
   - You have to restart the system to ensure macfuse modules work
- Download & install the version for MacOS Silicon (M1/M2) as mentioned here under MacOS : [CVMFS Readme](https://cvmfs.readthedocs.io/en/stable/cpt-quickstart.html)
- You might have to restart the system once again
- Then do the following step by step

>[!NOTE]
>You can club only certain commands in a single script. If you source the script, not everything will be executed due to shell spawning.

## Setting up cvmfs

```bash
# autofs is meant for linux systems
#sudo systemctl enable autofs
#sudo systemctl restart autofs

# Needs to be done only once
sudo mkdir -p /cvmfs/atlas.cern.ch
sudo mkdir -p /cvmfs/atlas-nightlies.cern.ch
sudo mkdir -p /cvmfs/atlas-condb.cern.ch
sudo mkdir -p /cvmfs/sft.cern.ch
sudo mkdir -p /cvmfs/sft-nightlies.cern.ch
sudo mkdir -p /cvmfs/unpacked.cern.ch
sudo mkdir -p /cvmfs/grid.cern.ch

# Needs to be done after each reboot
sudo mount -t cvmfs atlas.cern.ch /cvmfs/atlas.cern.ch
sudo mount -t cvmfs atlas-nightlies.cern.ch /cvmfs/atlas-nightlies.cern.ch
sudo mount -t cvmfs atlas-condb.cern.ch /cvmfs/atlas-condb.cern.ch
sudo mount -t cvmfs sft.cern.ch /cvmfs/sft.cern.ch
sudo mount -t cvmfs sft-nightlies.cern.ch /cvmfs/sft-nightlies.cern.ch
sudo mount -t cvmfs unpacked.cern.ch /cvmfs/unpacked.cern.ch
sudo mount -t cvmfs grid.cern.ch /cvmfs/grid.cern.ch

sudo cvmfs_config setup
#Some cvmfs diagnostic commands
 #sudo cvmfs_config chksetup
 #sudo cvmfs_config probe

export ATLAS_LOCAL_ROOT_BASE=/cvmfs/atlas.cern.ch/repo/ATLASLocalRootBase
alias setupATLAS='source ${ATLAS_LOCAL_ROOT_BASE}/user/atlasLocalSetup.sh'
#setupATLAS -q -c centos7
setupATLAS -q -c el9
lsetup git
```
Then create `default.local` in `/etc/cvmfs/` with the following content:

```bash
CVMFS_REPOSITORIES='atlas.cern.ch,atlas-nightlies.cern.ch,atlas-condb.cern.ch,grid.cern.ch,sft.cern.ch,sft-nightlies.cern.ch,unpacked.cern.ch '
CVMFS_CLIENT_PROFILE=single
CVMFS_HTTP_PROXY=DIRECT
```
## Grid Certificate Setup

Make sure the `~/.globus` folder from lxplus having `usercert.pem` & `userkey.pem` is stored in your MacBook's home directory `~/` and make sure the `.pem` files have appropriate octal `600` permissions for your local user.

### Downloading grid-security folder locally

Do the following, provided you see that `cvmfs` is mounted at `/cvmfs`

```bash
sudo cp -r /cvmfs/grid.cern.ch/etc/grid-security .
```
### Optional: Locally installing voms client and rucio

This assumes you have `anaconda3` installed using a required `python3.x` version
If not, use this resource: [Anaconda Python Distrib](https://anaconda.org)

Resources: 
- [Conda-forge VOMS](https://anaconda.org/conda-forge/voms)
- [Conda-forge Rucio](https://github.com/conda-forge/rucio-clients-feedstock)

```bash
conda config --add channels conda-forge
conda config --set channel_priority strict
conda install rucio-clients -c conda-forge
conda install voms -c conda-forge
```

## Set up Grid Proxy

Finally, to set up grid proxy, from within the container terminal environment, issue the following commands:

```bash
export X509_USER_CERT=~/.globus/usercert.pem
export X509_USER_KEY=~/.globus/userkey.pem
export X509_CERT_DIR=/cvmfs/grid.cern.ch/etc/grid-security/certificates
export X509_VOMS_DIR=/cvmfs/grid.cern.ch/etc/grid-security/vomsdir
export PATH=/cvmfs/grid.cern.ch/bin:$PATH
export LD_LIBRARY_PATH=/cvmfs/grid.cern.ch/lib:$LD_LIBRARY_PATH

lsetup rucio
lsetup panda
lsetup git
voms-proxy-init -voms atlas --vomses /cvmfs/grid.cern.ch/etc/grid-security/vomses/voms-atlas-auth.app.cern.ch.vomses
```
>[!NOTE]
>If you want to renew the proxy locally to access grid without setting up cvmfs, the optional steps become mandatory, when these commands are issued standlone, outside of the respective cvmfs environment.
>Also ensure that your conda environment is active
> To locally setup `grid-proxy` do the following below, assuming you have downloaded/copied `grid-security` under `/etc`
>`voms-proxy-init -voms atlas --vomses /etc/grid-security/vomses/voms-atlas-auth.app.cern.ch.vomses`

>[!NOTE]
>Finally to make rucio work locally, have these directives in `~/.rucio/rucio.cfg` for in container cvmfs use or `/opt/rucio/etc/rucio.cfg` , for standalone (optional) use. The `~/.rucio/rucio.cfg` usually overrides other root configs.

```bash
[client]
  rucio_host =  https://voatlasrucio-server-prod.cern.ch:443
  auth_host = https://atlas-rucio-auth.cern.ch:443
  ca_cert = /etc/grid-security/certificates
  auth_type = x509_proxy
  account = <cern username>
  client_cert = ~/.globus/usercert.pem
  client_key = ~/.globus/userkey.pem
```
>[!TIP]
>Optional: To be sure, you can also put the following lines in a script and source that script.
>```bash
>export RUCIO_ACCOUNT=<CERN_USERNAME>
>export RUCIO_AUTH_TYPE=x509_proxy
>export X509_CERT_DIR=/cvmfs/grid.cern.ch/etc/grid-security/certificates
>export X509_USER_CERT=$HOME/.globus/usercert.pem
>export X509_USER_KEY=$HOME/.globus/userkey.pem
>export X509_USER_PROXY=/tmp/x509up_u`id -u`
>```
