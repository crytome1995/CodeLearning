# Requirements

- Ubuntu servers # masters + # workers
- kubeadm installed

1. pre-instance

   - Createh the ssh key
   - attach NAT to public subnet
   - add private subnet route table to direct all internet bound traffic to NAT
   - associate route table with private subnet

2. Setup your masters

   - give each one the name master-#
   - assign each one to the private subnet

3. setup your workers
   - give each one the name worker-#
   - assign each one to the private subnet
