# MPI Overview
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; - Author: Bin Tan

MPI (Message Passing Interface) is a well known distributed computation standard for HPC for 
many years. One popular implementation is [Open MPI](https://www.open-mpi.org). In this tutorial, 
basic and fundamental concepts are discussed and several examples are developed to help
understand the concepts in depth. They are grouped into following 6 categories

- Communicator and Rank
- Group
- Inter-Communicator and Point-to-Point Communication
- Intra-Communicator and Collective Operations

### Communicator and Rank
In a distributed computation world, how to communicate among the processes is essential. 
In MPI, Communicator is the object representing the network which connects a group 
computation processes. The id of each process is given and differentiated by the rank. 

The default and also the biggest communicator is MPI_COMM_WORLD which contains all the
processes for an application. This default communicator is created when the mpirun command
is executed. For example,

```
$ mpirun -n 2 -host node1:1,node2:1 ./mpi_hello_world
```
This command shall create two processes. One is on the node1 and one is on the node2. Each 
process shall execute application mpi_hello_world. The following code shows how to access the MPI_COMM_WORLD
within the example_application

```cpp
    int nRanks;    // the size of the default communicator
    int myRank;    // the rank of this process in the default communicator

    MPI_init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &nRanks);
    MPI_Comm_rank(MPI_COMM_WORLD, &myRank);
    
```
After the rank is identified, example_application can know which process it is running with. The option
"-n 2" means mpirun shall create two processes and the option "-host node1:1,node2:1" tells mpirun to
create one process on each node1 and node2. Here is the code of the simple [Hello World example](./mpi_hello_world.cpp).

### Group
The default communicator, MPI_COMM_WORLD, contains all processes created by the mpirun command. For point-to-point communication,
it seems sufficient. However, for collective operation communication, in many situations, all processes  is not
preferrable. Therefore, a concept to capture a subset of all processes is desirable. The new concept is called group. 
A group is defined as an ordered set of process identifiers and each process associated with the group is assigned an additional
integer id as rank of the process in the group. This means that the process could have multiple ranks, for example, one is associated
with a group and another is in the default communicator, MPI_COMM_WORLD. 

There are two pre-defined groups, MPI_GROUP_EMPTY and MPI_GROUP_NULL. MPI_GROUP_EMPTY is a valid group with no processes 
and MPI_GROUP_NULL is an invalid group.

The key difference between group and communicator is that group is a conceptual representation of a set of processes and does not
support any communication functionality like a communicator. To allow processes within a group be able to communicate to each other
like a MPI communicator, a communicator shall be created with the group. The API to create a 
new MPI communicator associated with the group is

```c
     MPI_Comm_create_group(MPI_Comm comm, MPI_Group group, int tag, MPI_Comm* newconn)
```

One important thing in MPI is that a group can not be created from scratch. It has to be created (or formed) from an exist
group via group (aka set) operations. There is a base group created by the MPI, which is associated with the 
default communicator, MPI_COMM_GROUP. It can be retrieved by API,

```c
     MPI_Comm_group(MPI_Comm comm, MPI_Group *group)
```

The above two APIs reflect one important implementation fact which is that for a MPI communicator, there always is a group associated with it and
can be retrieved by MPI_Comm_group. However, for a group, there may not be a MPI communicator associated with it. If so, it has to
explicitly create one via MPI_Comm_create_group.

This is [an example](./mpi_group_example.cpp) to demonstrate above discussion about the group concept.

```
$ mpirun -n 6 -host gpt-1:3,gpt-2:3 ./mpi_group_example
The process 0 in sub group 0 with sub group rank 0 received broadcast value 2
The process 1 in sub group 0 with sub group rank 1 received broadcast value 2
The process 2 in sub group 1 with sub group rank 0 received broadcast value 3
The process 4 in sub group 2 with sub group rank 0 received broadcast value 4
The process 5 in sub group 2 with sub group rank 1 received broadcast value 4
The process 3 in sub group 1 with sub group rank 1 received broadcast value 3
-----------------------------------------------
Process 0 is not in the diff sub group
The process 2 in sub group 1 with sub group rank 0 and with new sub group rank 0 in sub group 0_1 union received broadcast value 101
Process 1 is not in the diff sub group
The process 4 in sub group 2 with sub group rank 0 and with new sub group rank 2 in sub group 0_1 union received broadcast value 101
The process 5 in sub group 2 with sub group rank 1 and with new sub group rank 3 in sub group 0_1 union received broadcast value 101
The process 3 in sub group 1 with sub group rank 1 and with new sub group rank 1 in sub group 0_1 union received broadcast value 101
```

### Inter-Communicator and Point-to-Point Communication

A communicator can be either an inter-communicator or intra-communicator but not both. An inter-communicator is a point-to-point
communication between processes in different groups. An inter-communicator can not be used for collective communication. 

To create an inter-communicator, two disjoint groups are created first. The group can be created with following API
based on the default group associated with the default communicator, MPI_COMM_WORLD,

```c
    int MPI_Group_incl(MPI_Group group, int n, const int ranks[], MPI_Group *newgroup)
```

Then, an inter-communicator can be created with following API,

```c
    int MPI_Intercomm_create_from_groups(MPI_Group local_group, int local_leader,
                                         MPI_Group remote_group, int remote_leader,
                                         const char* stringtag,
                                         MPI_Errhandler errhandler,
                                         MPI_Comm* newintercomm)
```

After the inter-communicator is created, the following point-to-point communication APIs can be used for sending and 
receiving data.

```c
    int MPI_Send(const void* buf, int count, MPI_Datatype datatype,
                 int dest, int tag, MPI_Comm comm)
    int MPI_Recv(void* buf, int count, MPI_Datatype datatype,
                 int source, int tag, MPI_Comm comm, MPI_Status *status)
```

The complete implemented [example](./mpi_inter_comm.cpp), two disjoint groups with world wide ranks
{0, 1, 2} and { 3, 4, 5} are created. After a sub group is created, each process shall be assigned a new 
rank with respect to the sub group. Therefore, each process shall have two ranks.

### Intra-Communicator and Collective Communication
An Intra-Communicator is able to suppor collective communication operation. Each collective communication operation 
invovles all member processes in the communicator. For example, MPI_Bcast API,
```c
    int MPI_Bcast(void* buffer, int count, MPI_Datatype datatype, int root, MPI_Comm comm)
```
shall broadcast the send buffer to all the processes which are the members of the intra-communicator, comm. 
This API is a blocking call untill all member processes finish the call. To facilitate the broadcast capability,
all member processes need to call this API with the same root and comm arguments. The root process is the process
which generates the original content of the buffer. It does not mean the root of the communicator or the lead
rank of the communicator. It only has an API call scope.

Another common used collective communication API is,
```c
    int MPI_Reduce(const void* sendbuf, void* recvbuf, int count, MPI_Datatype datatpye,
                   MPI_Op op, int root, MPI_Comm comm)
```
where MPI_Op has following pre-defined handle,
```c
     MPI_OP_NULL,
     MPI_MAX,
     MPI_MIN,
     MPI_SUN,
     MPI_PROD,
     MPI_LAND,
     MPI_BAND,
     MPI_LOR,
     MPI_BOR,
     MPI_LXOR,
     MPI_BXOR,
     MPI_MINLOC,
     MPI_MAXLOC,
     MPI_REPLACE
```
The reduce call is a global collective operation which perform elementwise op on the sendbuf from each process and write
the results into the recvbuf in the process whose rank is equal to the root argument. All the member processes of 
the communicator comm must call this API with the same count, datatype, op, root and comm arguments. The sendbuf
and recvbuf are per process basis. 

Further, the MPI_Op op can be customized with the following API for an application
```c
    int MPI_Op_create(MPI_User_function *f, int commute, MPI_Op *op)
```
where the MPI_User_function f must have the following signature,
```c
    void MPI_User_function(void *invec, void* inoutvec, int *len, MPI_Datatype *datatype).
```
[mpi_intra_comm.cpp](./mpi_intra_comm.cpp) defines a customized op to multiply a row of a matrix with a vector.
