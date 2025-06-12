## Understanding about linux capabilities
 privileged container system with root access gives you access to the host filesystem, kernel settings and processes.

## How capabilities affect Docker containers

## Privileged mode in Docker
eg. docker run -it ubuntu:latest bash                                                  [22:49:36]
root@704cd938492d:/# echo 1 > /proc/sys/net/ipv4/ip_forward
bash: /proc/sys/net/ipv4/ip_forward: Read-only file system
root@704cd938492d:/# exit
exit
rtvkiz:yahoo/ (master✗) $ docker run -it --privileged ubuntu:latest bash                                     [22:54:14]
root@9ec8a35ae8b4:/# echo 1 > /proc/sys/net/ipv4/ip_forward


## What happens when we are running as privileged container
- So what I learned is - all the process in the container will have the bounding capabilities as the maximum value, and its more than the admin value
admin containers have more than the normal ones. This would be good example of printing out capabilities with different normal docker capabilities, admin and privileged. Using tool to conver them to another value

create containers with admin, privileged and normal
Run a process and check the capabilities for each one of them