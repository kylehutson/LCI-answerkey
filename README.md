# LCI-answerkey
## Background for both Slurm and Ceph:
- Vanilla Rocky 8 minimal install.
- IP addresses configured with a gateway that lets you out to the Internet
- DNS for all hosts configured
- Passwordless SSH enabled from all hosts to all other hosts
  - ssh to head
  - ssh-keygen
  - for x in head compute{1..2} storage{1..4}; do ssh-copy-id $x ; done
  - for x in head compute{1..2} storage{1..4}; do scp ~/.ssh/* ${x}:.ssh/ ; done
