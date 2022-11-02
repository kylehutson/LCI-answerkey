# LCI-answerkey
## Background for both Slurm and Ceph:
- Vanilla Rocky 8 minimal install.
- IP addresses configured with a gateway that lets you out to the Internet
- DNS for all hosts configured
- Passwordless SSH enabled from all hosts to all other hosts
  - `ssh root@head`
  - `[root@head ~]#ssh-keygen`
  - `[root@head ~]# for x in head compute{1..2} storage{1..4}; do ssh-copy-id $x ; done`
  - `[root@head ~]# for x in head compute{1..2} storage{1..4}; do scp ~/.ssh/* ${x}:.ssh/ ; done`
## Slurm
- Install and configure munge
  - Set the config-manager so it can find munge
    - `[root@head ~]# for x in head compute{1..2}; do ssh $x yum config-manager --set-enabled powertools; done`
  - Install munge everywhere
    - `[root@head ~]# for x in head compute{1..2}; do ssh $x yum install -y munge; done`
  - Create the munge key on head
    - `[root@head ~]# create-munge-key`
  - Copy the munge key to the compute nodes and set correct permissions
    - `[root@head ~]# for x in compute{1..2}; do scp /etc/munge/munge.key ${x}:/etc/munge/ ; done`
    - `[root@head ~]# for x in compute{1..2}; do ssh $x chown munge:munge /etc/munge/munge.key ; done`
  - Start munge and enable it everywhere
    - `[root@head ~]# for x in head compute{1..2}; do ssh $x systemctl enable munge ; done`
    - `[root@head ~]# for x in head compute{1..2}; do ssh $x systemctl start munge ; done`
  - Get my favorite downloading tool so I can use it in the next step
    - `[root@head ~]# yum install -y wget`
  - Download Slurm into its own directory
    - `[root@head ~]# mkdir slurm`
    - `[root@head ~]# cd slurm`
    - `[root@head slurm]# wget https://download.schedmd.com/slurm/slurm-22.05.5.tar.bz2`
  - Get the tools to install Slurm
    - `[root@head slurm]# yum -y install rpm-build munge-devel pam-devel perl python3 readline-devel gcc`
  - The Slurm installer will fail if the MariaDB libraries aren't installed first, so we're going to install that server now
    - `[root@head slurm]# yum install -y mariadb-server mariadb-devel`
  - Build the Slurm RPM
    - `[root@head slurm]# rpmbuild -ta slurm*.tar.bz2`
  - Go the the directory where the RPMs were built and install the ones we need on the headnode
    - `[root@head slurm]# cd ~/rpmbuild/RPMS/x86_64/`
    - `[root@head x86_64]# yum install -y slurm-slurmctld-22.05.5-1.el8.x86_64.rpm slurm-slurmdbd-22.05.5-1.el8.x86_64.rpm slurm-22.05.5-1.el8.x86_64.rpm`
  - Configure MariaDB. Here I'm going to give the password 'mysecret'
    - `[root@head x86_64]# systemctl start mariadb`
    - `[root@head x86_64]# systemctl enable mariadb`
    - `[root@head x86_64]# mysql_secure_installation`
      - Follow the prompts with no existing password, 'y' to set the new password, enter 'mysecret' twice (it will not show on the screen), 'y' to remove anonymous users, 'y' to disallow remote root login, 'y' to remove test database, and 'y' to reload privilege tables
  - Configure a slurm database. Here I'm going to use the password 'slurmpass'
    - `[root@head x86_64]# mysql -p`
      - Enter the password you created above
    - `MariaDB [(none)]> grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'slurmpass' with grant option;`
    - `MariaDB [(none)]> create database slurm_acct_db;`
    - `MariaDB [(none)]> \q`
  - Create an appropriate slurmdbd.conf`
    - `[root@head x86_64]# mkdir /etc/slurm`
    - `[root@head x86_64]# cat << EOF > /etc/slurm/slurmdbd.conf
> LogFile=/var/log/slurm/slurmdbd.log
> DbdHost=head
> DbdPort=6819
> SlurmUser=slurm
> StorageHost=head
> StoragePass=slurmpass
> StorageLoc=slurm_acct_db
> EOF`

