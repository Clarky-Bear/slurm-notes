# slurm-notes

Install steps

1:Hostnames/Networking
2:Install Database
3:Create Users
4:Install Munge
5:Install Slurm
6:

(Quick installs)

yum install nano gcc epel-release rpmbuild git

(git clone <insert url of files to download>)

*********************************************************
1:Hostnames and Networking

(Set hostnames/IP addressing and then bring interfaces up.)

yum install nano

nano /etc/hostname

add hostname (Virt1/Virt2/Virt3)

nano /etc/hosts
add hosts
192.168.21.128 Virt1
192.168.21.129 Virt2
192.168.21.130 Virt3

(DNS server)
nano /etc/resolv.conf - add dns server (nameserver 8.8.8.8)

systemctl restart network

hostnamectl status (Display hsotname info)

1.1*******************set ip addresses

cd /etc/sysconfig/network-scripts/
nano ifcg-eth0 (Or whatever interface is named)

Change DHCP to static then below add
IPADDR=ip
NETMASK=mask
GATEWAY=ip
save/exit


systemctl restart networkt
ifconfig/ ip addr to verify







******************************************************
2:Install database software (On server)


yum install mariadb mariadb-server mariadb-devel -y

**********************************************************************
3: Create user groups
(Ensure each UID/GID is same across all nodes)

#!/bin/bash
export MUNGEUSER=991
groupadd -g $MUNGEUSER munge
useradd  -m -c "Mr Munge" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge

export SLURMUSER=992
groupadd -g $SLURMUSER slurm
useradd  -m -c "Slurm" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm

export SLURMUSER1=993
useradd  -m -c "User1" -d /home/user1 -u $SLURMUSER1 -g slurm  -s /bin/bash user1

export SLURMUSER2=994
useradd  -m -c "User2" -d /home/user2 -u $SLURMUSER2 -g slurm  -s /bin/bash user2

tail /etc/passwd (Verify users)



*************************************************************************
4: Install Munge

yum install munge munge-libs munge-devel -y (All nodes)

Create Key(On Server only)

yum install rng-tools -y
(Feeds random data to kernel random number pool)

rngd -r /dev/urandom

/usr/sbin/create-munge-key -r

(User data duplicator, set input file and outputfile)

dd if=/dev/urandom bs=1 count=1024 > /etc/munge/munge.key

(Change ownership and file access)
chown munge: /etc/munge/munge.key
chmod 400 /etc/munge/munge.key

(Copy munge key to all nodes)
scp /etc/munge/munge.key mradmin@Virt2:/etc/munge (If permission denied simply send to home folder and copy manually from node for now)

(SSH into each node, verfy key is in correct directory and do following:)

chown -R munge: /etc/munge/ /var/log/munge/
chmod 0700 /etc/munge/ /var/log/munge/

(Start munge)
systemctl enable munge
systemctl start munge

(Test munge
munge -n
munge -n | unmunge
munge -n | ssh Virt1 unmunge
remunge)



4.5***************************************************
Database setup

systemctl enable mariadbnano 
systemctl start mariadb

mysql_sercure_isntallation

mysql -u root -p
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('1234');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;




*******************************************************
5:Install SLURM 

**NTP (all nodes)

yum install ntp -y
chkconfig ntpd on
ntpdate pool.ntp.org
systemctl start ntpd

**Install required packages(All nodes)

yum install perl-devel openssl openssl-devel pam-devel numactl numactl-devel 
hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel 
man2html libibmad libibumad -y

**Download latest Slurm
wget http://www.schedmd.com/downloads/latest/slurm-17.02.9.tar.bz2

**Build rpms. Ensure all packages have been isntalled prior to attempting build

rpmbuild -ta slurm-17.02.7.tar.bz2

**TRanfer RPM files to nodes
(Eitehr use SCP or quickly setup NFS on each node)

cd /root/rpmbuild/RPM/x86_64/

scp *.rpm mradmin@Virt2:/home/mradmin/slurm/
scp *.rpm mradmin@Virt3:/home/mradmin/slurm/

yum --nogpgcheck localinstall *.rpm

Grab and configure slurm.conf
Grab and configure slurmdbd.conf



(Copy slurm.conf to all nodes (If permission denied simply go to home)
scp slurm.conf mradmin@Virt2:/etc/slurm/slurm.conf

Put slurmdbd.conf into /etc/slurm/


**Create directoies for server

mkdir /var/spool/slurmctld
chown slurm: /var/spool/slurmctld
chmod 755 /var/spool/slurmctld
touch /var/log/slurmctld.log
chown slurm: /var/log/slurmctld.log
touch /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log /var/log/slurmdbd
chown slurm: /var/log/slurm_jobacct.log /var/log/slurm_jobcomp.log

**Create Directories for nodes

mkdir /var/spool/slurmd
chown slurm: /var/spool/slurmd
chmod 755 /var/spool/slurmd
touch /var/log/slurmd.log
chown slurm: /var/log/slurmd.log

slurmd -C (VErify slurm conf is fine)

**Firewall config before starting

(Turn off firewall on nodes/server is having too many issues)
systemctl stop firewalld
systemctl disable firewalld





***************Start Compute nodes
systemctl enable slurmd.service
systemctl start slurmd.service
systemctl status slurmd.service

**Start Server

systemctl enable slurmdbd
systemctl start slurmdbd

*Do accounting then start bloody slurmctld*

systemctl enable slurmctld.service
systemctl start slurmctld.service
systemctl status slurmctld.service



sinfo

Accounting/DB
sacctmgr add cluster virt
sacctmgr add account root Cluster=virt
sacctmgr add account testusers Cluster=virt
sacctmgr add user test1 DefaultAccount=testusers
sacctmgr add user test2 DefaultAccount=testusers

**** TEST
scontrol show nodes

****Test Command
srun -N2 hostname
sbatch -N2 batch.sh











