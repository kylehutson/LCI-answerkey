# LCI-answerkey
## Background for both Slurm and Ceph:
- Vanilla Rocky 8 minimal install.
- IP addresses configured with a gateway that lets you out to the Internet
- DNS for all hosts configured
- Passwordless SSH enabled from all hosts to all other hosts
 ```
$ ssh root@head
[root@head ~]# ssh-keygen
[root@head ~]# for x in head compute{1..2} storage{1..4}; do ssh-copy-id $x ; done
[root@head ~]# for x in head compute{1..2} storage{1..4}; do scp ~/.ssh/* ${x}:.ssh/ ; done
```
## Slurm
### Munge
Set the config-manager so it can find munge
```
[root@head ~]# for x in head compute{1..2}; do ssh $x yum config-manager --set-enabled powertools; done
```
Install munge everywhere
```
[root@head ~]# for x in head compute{1..2}; do ssh $x yum install -y munge; done
```
Create the munge key on head
```
[root@head ~]# create-munge-key
```
Copy the munge key to the compute nodes and set correct permissions
```
[root@head ~]# for x in compute{1..2}; do scp /etc/munge/munge.key ${x}:/etc/munge/ ; done
[root@head ~]# for x in compute{1..2}; do ssh $x chown munge:munge /etc/munge/munge.key ; done
```
Start munge and enable it everywhere
```
[root@head ~]# for x in head compute{1..2}; do ssh $x systemctl enable munge ; done
[root@head ~]# for x in head compute{1..2}; do ssh $x systemctl start munge ; done
```
### Slurm installers
Get my favorite downloading tool so I can use it in the next step (You could use the built-in 'curl', but you'll have to adjust your own downloads accordingly)
```
[root@head ~]# yum install -y wget
```
Download Slurm into its own directory
```
[root@head ~]# mkdir slurm
[root@head ~]# cd slurm
[root@head slurm]# wget https://download.schedmd.com/slurm/slurm-22.05.5.tar.bz2
```
Get the tools to install Slurm
```
[root@head slurm]# yum -y install rpm-build munge-devel pam-devel perl python3 readline-devel gcc
```
The Slurm installer will fail if the MariaDB libraries aren't installed first, so we're going to install that server now
```
[root@head slurm]# yum install -y mariadb-server mariadb-devel
```
Build the Slurm RPM
```
[root@head slurm]# rpmbuild -ta slurm*.tar.bz2
```
Go the the directory where the RPMs were built and install the ones we need on the headnode
```
[root@head slurm]# cd ~/rpmbuild/RPMS/x86_64/
[root@head x86_64]# yum install -y slurm-slurmctld-22.05.5-1.el8.x86_64.rpm slurm-slurmdbd-22.05.5-1.el8.x86_64.rpm slurm-22.05.5-1.el8.x86_64.rpm
```
### MariaDB
Configure MariaDB. Here I'm going to give the password 'mysecret'
```
[root@head x86_64]# systemctl start mariadb
[root@head x86_64]# systemctl enable mariadb
[root@head x86_64]# mysql_secure_installation
```
 - Follow the prompts with no existing password, `y` to set the new password, enter `mysecret` twice (it will not show on the screen), `y` to remove anonymous users, `y` to disallow remote root login, `y` to remove test database, and `y` to reload privilege tables


Configure a slurm database. Here I'm going to use the password 'slurmpass'
```
[root@head x86_64]# mysql -p
```
 - Enter the password you created above
```
MariaDB [(none)]> grant all on slurm_acct_db.* TO 'slurm'@'%' identified by 'slurmpass' with grant option;
MariaDB [(none)]> create database slurm_acct_db;
MariaDB [(none)]> flush privileges;
MariaDB [(none)]> \q
```
### SlurmDBD
**Note:** In the real-world, you probably want the slurmdbd to be on it's own host. You will want to change these parameters accordingly. See https://slurm.schedmd.com/slurmdbd.conf.html

Create an appropriate slurmdbd.conf
```
[root@head x86_64]# for x in head compute{1..2}; do ssh $x mkdir /etc/slurm ; done
[root@head x86_64]# cat << EOF > /etc/slurm/slurmdbd.conf
> LogFile=/var/log/slurm/slurmdbd.log
> DbdHost=head
> DbdPort=6819
> SlurmUser=slurm
> StorageHost=head
> StoragePass=slurmpass
> StorageLoc=slurm_acct_db
> StorageType=accounting_storage/mysql
> EOF
```
We need to create a 'slurm' user for running these processes. You don't need to manually set the gid and uid, but it ensures they are consistent
```
[root@head x86_64]# for x in head compute{1..2}; do ssh $x 'groupadd --gid 3000 slurm' ; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x 'useradd --uid 3000 slurm -g slurm --shell /bin/nologin --no-create-home --comment "Slurm User"' ; done
```
Copy the slurmdbd.conf everywhere and give appropriate permissions
```
[root@head x86_64]# for x in compute{1..2}; do scp /etc/slurm/slurmdbd.conf ${x}:/etc/slurm/ ; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x chown slurm:slurm /etc/slurm/slurmdbd.conf ; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x chmod go-rwx /etc/slurm/slurmdbd.conf ; done
```
Create the default logging directory and change permissions accordingy
```
[root@head x86_64]# mkdir /var/log/slurm
[root@head x86_64]# chown slurm:slurm /var/log/slurm
```
Start the slurmdbd service and enable it for auto start on reboot
```
[root@head x86_64]# systemctl start slurmdbd
[root@head x86_64]# systemctl enable slurmdbd
```
### SlurmCtlD
We need to create a slurm.conf - Use https://slurm.schedmd.com/configurator.html
```
[root@head x86_64]# cat << EOF > /etc/slurm/slurm.conf
> ClusterName=lcicluster
> SlurmctldHost=head
> AccountingStorageEnforce=associations,limits
> AccountingStorageHost=head
> AccountingStorageType=accounting_storage/slurmdbd
> MpiDefault=none
> SlurmctldPort=6817
> ProctrackType=proctrack/cgroup
> ReturnToService=1
> SlurmdPidFile=/var/run/slurmd.pid
> SlurmdPort=6818
> SlurmdSpoolDir=/var/spool/slurmd
> SlurmUser=slurm
> StateSaveLocation=/var/spool/slurmctld
> SlurmctldPidFile=/var/run/slurmctld.pid
> SlurmctldPort=6817
> SlurmdPidFile=/var/run/slurmd.pid
> SlurmdPort=6818
> SlurmdSpoolDir=/var/spool/slurmd
> SlurmUser=slurm
> StateSaveLocation=/var/spool/slurmctld
> SwitchType=switch/none
> TaskPlugin=task/affinity
> InactiveLimit=0
> KillWait=30
> MinJobAge=300
> SlurmctldTimeout=120
> SlurmdTimeout=300
> Waittime=0
> SchedulerType=sched/backfill
> SelectType=select/cons_tres
> JobCompType=jobcomp/none
> JobAcctGatherFrequency=30
> JobAcctGatherType=jobacct_gather/none
> SlurmctldDebug=info
> SlurmctldLogFile=/var/log/slurmctld.log
> SlurmdDebug=info
> SlurmdLogFile=/var/log/slurmd.log
> NodeName=compute[1-2] CPUs=2 State=UNKNOWN
> PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP
> EOF
```
Now copy that everywhere and set perms
```
[root@head x86_64]# for x in compute{1..2}; do scp /etc/slurm/slurm.conf ${x}:/etc/slurm/ ; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x chmod 644 /etc/slurm/slurm.conf  ; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x chown slurm:slurm /etc/slurm/slurm.conf  ; done
```
Create the spool directory we specified in slurm.conf and set perms
```
[root@head x86_64]# mkdir /var/spool/slurmctld
[root@head x86_64]# chown slurm:slurm /var/spool/slurmctld/
```
Start and enable SlurmCtlD
```
[root@head x86_64]# systemctl start slurmctld
[root@head x86_64]# systemctl enable slurmctld
```
### Slurm nodes
We already have the slurm.conf installed and configured, so we just needto copy the RPMs and install them.

Note that I'm still in the correct rpmbuild directory. If you have changed directories while installing, you'll need to adjust accordingly.
```
[root@head x86_64]# for x in compute{1..2}; do scp slurm-22.05.5-1.el8.x86_64.rpm slurm-slurmd-22.05.5-1.el8.x86_64.rpm ${x}: ; done
[root@head x86_64]# for x in compute{1..2}; do scp slurm-22.05.5-1.el8.x86_64.rpm slurm-slurmd-22.05.5-1.el8.x86_64.rpm ${x}: ; done
```
We need to have a /etc/slurm/cgroup.conf, so we'll just create an empty file there
```
[root@head x86_64]# for x in compute{1..2}; do ssh $x touch /etc/slurm/cgroup.conf ; done
```
FirewallD will prevent slurmd from communicating. The *right* way is to correctly issue firewall-cmd rules. For this exercise, I'm just going to disable it altogether. (And, for what it's worth, my entire cluster is behind a firewall, so I have it disabled there also.)
```
[root@head x86_64]# for x in head compute{1..2}; do ssh $x systemctl stop firewalld; done
[root@head x86_64]# for x in head compute{1..2}; do ssh $x systemctl disable firewalld; done
```
With all the prerequisites in place, we can now start and enable slurmd
```
[root@head x86_64]# for x in compute{1..2}; do ssh $x systemctl start slurmd ; done
[root@head x86_64]# for x in compute{1..2}; do ssh $x systemctl enable slurmd ; done
```
Now test...
```
cd
[root@head ~]# srun --pty /bin/bash
[root@compute1 ~]# hostname
```
The system should respond with 'compute1'
```
[root@compute1 ~]# exit
```
Everything is running! Hooray!

...Except now we have to create users and groups and tweak things

To keep things straight, I'm going with the mnemonic approach
Alexander and Amelia will belong to the Anatomy group
Benjamin and Brooklyn will belong to the Biology group
Carter and Charlotte will belong to the Chemistry group
Daniel and Delilah will belong to the Diagnostics group
```
[root@head ~]# for x in head compute{1..2}; do ssh $x 'for y in alexander amelia benjamin brooklyn carter charlotte daniel delilah; do useradd $y; done'; done
```
Now we need to associate these with groups. We start by adding the cluster in SAcctMgr
```
[root@head ~]# sacctmgr add cluster lcicluster
```
Add the departments
```
[root@head ~]# for x in anatomy biology chemistry diagnostics ; do sacctmgr create account $x ; done
```
Note that you will have to confirm with a `y` for each of these

Now we add users to those accounts
```
[root@head ~]# sacctmgr create user name=alexander DefaultAccount=anatomy
[root@head ~]# sacctmgr create user name=amelia DefaultAccount=anatomy
[root@head ~]# sacctmgr create user name=benjamin DefaultAccount=biology
[root@head ~]# sacctmgr create user name=brooklyn DefaultAccount=biology
[root@head ~]# sacctmgr create user name=carter DefaultAccount=chemistry
[root@head ~]# sacctmgr create user name=charlotte DefaultAccount=chemistry
[root@head ~]# sacctmgr create user name=daniel DefaultAccount=diagnostics
[root@head ~]# sacctmgr create user name=delilah DefaultAccount=diagnostics
```
Again, you will have to confirm each of these with a `y`

