# Introduction

This heat template allows you to easily deploy a mpi cluster on OpenStack.

# Quick start
1) Clone this repository :

`git clone https://github.com/frgaudet/openstack-heat-mpi.git`

2) Source your OpenStack environment file :

`source openrc.sh`

3) Prepare your parameters :

	* key_name: Name of key-pair to be used
	* image_id: Server image
	* net_id: Private network id
	* name: Prefix for all your instances's name

4) Launch a Cluster :

```
cd openstack-heat-mpi
heat stack-create -f mpi.yaml \
	-e lib/env.yaml \
	-P "key_name=fgaudet-key;image_id=1b115984-32b9-4485-a664-26229db439fa;net_id=dev-net;name=fgaudet-mpi" mpi-stack
```

# Parameters

There is at least 1 node, so if you specify count=4, then you'll have 5 nodes.

For example, create a 5 nodes cluster, with a m1.xlarge flavor :

```
heat stack-create -f mpi.yaml \
	-e lib/env.yaml \
	-P "count=4;flavor=m1.xlarge;key_name=fgaudet-key;image_id=1b115984-32b9-4485-a664-26229db439fa;net_id=dev-net;name=fgaudet-mpi" mpi-stack
```

# Check the master

Connect to your master, using the floating IP which has been automatically set up.

`ssh <username>@<IP>`

Then login with the mpi user (via root):

```
ubuntu@fgaudet-mpi:~$ sudo su -
root@fgaudet-mpi:~# su - mpi
```

and check your master :

```
wget http://svn.open-mpi.org/svn/ompi/tags/v1.6-series/v1.6.4/examples/connectivity_c.c
mpicc connectivity_c.c -o connectivity
mpirun ./connectivity
```
You should see the following output :
```
--------------------------------------------------------------------------
[[30818,1],0]: A high-performance Open MPI point-to-point messaging module
was unable to find any relevant network interfaces:

Module: OpenFabrics (openib)
  Host: fgaudet-mpi

Another transport will be used instead, although this may result in
lower performance.
--------------------------------------------------------------------------
Connectivity test on 1 processes PASSED.

```

Ok now we'll have a running mpi master, let's try out now a real application !

# Run simple ray-tracing application

The following steps were inspired from [here](https://support.rackspace.com/how-to/high-performance-computing-cluster-in-a-cloud-environment/ "Rackspace How to")

## Get application
wget http://jedi.ks.uiuc.edu/~johns/raytracer/files/0.99b2/tachyon-0.99b2.tar.gz -O /tmp/tachyon.tar.gz

## Deploy on every node
```
for i in $(cat mpi_hosts)
do
echo "Copying to ${i}"
scp -r /tmp/tachyon.tar.gz ${i}:/home/mpi/
echo "Compiling tachyon on ${i}"
ssh ${i} "tar -zxf /home/mpi/tachyon.tar.gz ;cd /home/mpi/tachyon/unix;make linux-mpi"
done
```

## Run the application on one node
```
cd ~/tachyon/compile/linux-mpi
./tachyon ../../scenes/teapot.dat -format BMP
```
You get the following output :

```
--------------------------------------------------------------------------
[[31156,1],0]: A high-performance Open MPI point-to-point messaging module
was unable to find any relevant network interfaces:

Module: OpenFabrics (openib)
  Host: fgaudet-mpi

Another transport will be used instead, although this may result in
lower performance.
--------------------------------------------------------------------------
Tachyon Parallel/Multiprocessor Ray Tracer   Version 0.99
Copyright 1994-2011,    John E. Stone <john.stone@gmail.com>
------------------------------------------------------------
Scene Parsing Time:     0.0192 seconds
Scene contains 2330 objects.
Preprocessing Time:     0.0026 seconds
Rendering Progress:       100% complete
  Ray Tracing Time:     0.7833 seconds
    Image I/O Time:     0.0061 seconds
```

## Run with 4 nodes
```
mpirun -np 4 --hostfile ~/mpi_hosts ./tachyon ../../scenes/teapot.dat -format BMP
```

You get the following output :

```
Warning: Permanently added '10.0.0.96' (ECDSA) to the list of known hosts.
Warning: Permanently added '10.0.0.95' (ECDSA) to the list of known hosts.
Warning: Permanently added '10.0.0.97' (ECDSA) to the list of known hosts.
--------------------------------------------------------------------------
[[31165,1],0]: A high-performance Open MPI point-to-point messaging module
was unable to find any relevant network interfaces:

Module: OpenFabrics (openib)
  Host: fgaudet-mpi

Another transport will be used instead, although this may result in
lower performance.
--------------------------------------------------------------------------
Tachyon Parallel/Multiprocessor Ray Tracer   Version 0.99
Copyright 1994-2011,    John E. Stone <john.stone@gmail.com>
------------------------------------------------------------
Scene Parsing Time:     0.0190 seconds
Scene contains 2330 objects.
Preprocessing Time:     0.0027 seconds
Rendering Progress:       100% complete
  Ray Tracing Time:     0.2543 seconds
    Image I/O Time:     0.0050 seconds
```
