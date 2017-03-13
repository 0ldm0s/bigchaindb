Run a BigchainDB Node in a Kubernetes Cluster
=============================================

Assuming you already have a `Kubernetes <https://kubernetes.io/>`_
cluster up and running, this page describes how to run a
BigchainDB node in it.


Step 1: Install kubectl
-----------------------

kubectl is the Kubernetes CLI.
If you don't already have it installed,
then see the `Kubernetes docs to install it
<https://kubernetes.io/docs/user-guide/prereqs/>`_.


Step 2: Configure kubectl
-------------------------

The default location of the kubectl configuration file is ``~/.kube/config``.
If you don't have that file, then you need to get it.

**Azure.** If you deployed your Kubernetes cluster on Azure
using the Azure CLI 2.0 (as per :doc:`our template <template-kubernetes-azure>`),
then you can get the ``~/.kube/config`` file using:

.. code:: bash

   $ az acs kubernetes get-credentials \
   --resource-group <name of resource group containing the cluster> \
   --name <ACS cluster name>


Step 3: Create Storage Classes
------------------------------

MongoDB needs somewhere to store its data persistently,
outside the container where MongoDB is running.

The official MongoDB Docker container exports two volume mounts with correct
permissions from inside the container:


* The directory where the mongod instance stores its data - ``/data/db``,
  described at `storage.dbpath <https://docs.mongodb.com/manual/reference/configuration-options/#storage.dbPath>`_.

* The directory where mongodb instance stores the metadata for a sharded
  cluster - ``/data/configdb/``, described at
  `sharding.configDB <https://docs.mongodb.com/manual/reference/configuration-options/#sharding.configDB>`_.


Explaining how Kubernetes handles persistent volumes,
and the associated terminology,
is beyond the scope of this documentation;
see `the Kubernetes docs about persistent volumes
<https://kubernetes.io/docs/user-guide/persistent-volumes>`_.

The first thing to do is create the Kubernetes storage classes.
We will accordingly create two storage classes and persistent volume claims in
Kubernetes.


**Azure.** First, you need an Azure storage account.
If you deployed your Kubernetes cluster on Azure
using the Azure CLI 2.0
(as per :doc:`our template <template-kubernetes-azure>`),
then the `az acs create` command already created two
storage accounts in the same location and resource group
as your Kubernetes cluster.
Both should have the same "storage account SKU": ``Standard_LRS``.
Standard storage is lower-cost and lower-performance.
It uses hard disk drives (HDD).
LRS means locally-redundant storage: three replicas
in the same data center.

Premium storage is higher-cost and higher-performance.
It uses solid state drives (SSD).
At the time of writing,
when we created a storage account with SKU ``Premium_LRS``
and tried to use that,
the PersistentVolumeClaim would get stuck in a "Pending" state.
For future reference, the command to create a storage account is
`az storage account create <https://docs.microsoft.com/en-us/cli/azure/storage/account#create>`_.


Get the file ``mongo-sc.yaml`` from GitHub using:

.. code:: bash

   $ wget https://raw.githubusercontent.com/bigchaindb/bigchaindb/master/k8s/mongodb/mongo-sc.yaml

You may want to update the ``parameters.location`` field in both the files to
specify the location you are using in Azure.


Create the required storage classes using

.. code:: bash

   $ kubectl apply -f mongo-sc.yaml


You can check if it worked using ``kubectl get storageclasses``.

Note that there is no line of the form
``storageAccount: <azure storage account name>``
under ``parameters:``. When we included one
and then created a PersistentVolumeClaim based on it,
the PersistentVolumeClaim would get stuck
in a "Pending" state.
Kubernetes just looks for a storageAccount
with the specified skuName and location.


Step 4: Create Persistent Volume Claims
---------------------------------------

Next, we'll create two PersistentVolumeClaim objects ``mongo-db-claim`` and
``mongo-configdb-claim``.

Get the file ``mongo-pvc.yaml`` from GitHub using:

.. code:: bash

   $ wget https://raw.githubusercontent.com/bigchaindb/bigchaindb/master/k8s/mongodb/mongo-pvc.yaml

Note how there's no explicit mention of Azure, AWS or whatever.
``ReadWriteOnce`` (RWO) means the volume can be mounted as
read-write by a single Kubernetes node.
(``ReadWriteOnce`` is the *only* access mode supported
by AzureDisk.)
``storage: 20Gi`` means the volume has a size of 20
`gibibytes <https://en.wikipedia.org/wiki/Gibibyte>`_.

You may want to update the ``spec.resources.requests.storage`` field in both
the files to specify a different disk size.

Create the required Persistent Volume Claims using:

.. code:: bash

   $ kubectl apply -f mongo-pvc.yaml


You can check its status using: ``kubectl get pvc -w``

Initially, the status of persistent volume claims might be "Pending"
but it should become "Bound" fairly quickly.


Step 5: Create the ConfigMap - Optional
---------------------------------------

This step is only required if you are planning to set up a cross datacenter
MongoDB cluster replica set. If you are planning to run multiple instances of
BigchainDB and MongoDB in the same datacenter, you may skip this step and move
to :doc:`Step 6 <Step 6: Run MongoDB as a StatefulSet>`.


MongoDB reads the local hosts file while bootstrapping a replica set.
We create a ConfigMap with the FQDN of the MongoDB instance and populate the
local hosts file with this value so that a replica set can be created
seamlessly.

Get the file ``mongo-cm.yaml`` from GitHub using:

.. code:: bash

   $ wget https://raw.githubusercontent.com/bigchaindb/bigchaindb/master/k8s/mongodb/mongo-cm.yaml

You may want to update the ``data.fqdn`` field in the file before applying it.
This name should resolve to the load balancer (or a HA instance) in you cluster
which is going to frontend your MongoDB instance.

If you are using Kubernetes on ACS, you can select a unique name here and we
can create the DNS A record for this in a later step as given below.
You can also use other DNS providers to supply the A record.


Create the required ConfigMap using:

.. code:: bash

   $ kubectl apply -f mongo-cm.yaml


You can check its status using: ``kubectl get cm``



Now we are ready to run MongoDB and BigchainDB on our Kubernetes cluster.

Step 6: Run MongoDB as a StatefulSet
------------------------------------

Get the file ``mongo-ss.yaml`` from GitHub using:

.. code:: bash

   $ wget https://raw.githubusercontent.com/bigchaindb/bigchaindb/master/k8s/mongodb/mongo-ss.yaml


Note how the MongoDB container uses the ``mongo-db-claim`` and the
``mongo-configdb-claim`` PersistentVolumeClaims for its ``/data/db`` and
``/data/configdb`` diretories (mount path). Note also that we use the pod's
``securityContext.capabilities.add`` specification to add the ``FOWNER``
capability to the container.

That is because MongoDB container has the user ``mongodb``, with uid ``999``
and group ``mongodb``, with gid ``999``.
When this container runs on a host with a mounted disk, the writes fail when
there is no user with uid ``999``.

To avoid this, we use the Docker feature of ``--cap-add=FOWNER``.
This bypasses the uid and gid permission checks during writes and allows data
to be persisted to disk.
Refer to the
`Docker doc <https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities>`_
for details.

As we gain more experience running MongoDB in testing and production, we will
tweak the ``resources.limits.cpu`` and ``resources.limits.memory``.
We will also stop exposing port ``27017`` globally and/or allow only certain
hosts to connect to the MongoDB instance in the future.

Create the required StatefulSet using:

.. code:: bash

   $ kubectl apply -f mongo-ss.yaml

You can check its status using the commands ``kubectl get statefulsets -w``
and ``kubectl get svc -w``

Note that you may have to wait for upto 10 minutes wait for disk to be created
and attached on the first run. The pod can fail several time with the message
specifying that the timeout for disk mount has exceeded.


Step 7: Initialize a MongoDB Replica Set
----------------------------------------

This step is only required if you are planning to set up a cross datacenter
MongoDB cluster replica set. If you are planning to run multiple instances of
BigchainDB and MongoDB in the same datacenter, you may skip this step and move
to :doc:`Step 9 <Step 9: Run BigchainDB as a Deployment>`.

Login to the running MongoDB instance and access the mongo shell using:

.. code:: bash
   
   $ kubectl exec -it mdb-0 -c mongodb -- /bin/bash
   $ mongo --port 27017

We initialize the replica set by using the ``rs.initialize()`` command from the
mongo shell, the syntax for which is:

.. code-block:: text
    rs.initiate({ 
        _id : "<replica-set-name", members: [
        { 
            _id : 0,
            host : "<fqdn of this instance>:<port number>"
        } ]
    })

For example, an init command might look like:

.. code:: bash
   
   > rs.initiate({ _id : "bigchain-rs", members: [ { _id : 0, host : "bdb-cluster-0.westeurope.cloudapp.azure.com:27017" } ] })


You should see changes in the mongo shell prompt from ``>`` to``bigchain-rs:OTHER>` to `bigchain-rs:SECONDARY>` to finally ``bigchain-rs:PRIMARY>``.

You can use the ``rs.conf()`` and the ``rs.status()`` commands to check the
detailed replica set configuration now.


Step 8: Create a DNS record - Optional
--------------------------------------

This step is only required if you are planning to set up a cross datacenter
MongoDB cluster replica set. If you are planning to run multiple instances of
BigchainDB and MongoDB in the same datacenter, you may skip this step and move
to :doc:`Step 9 <Step 9: Run BigchainDB as a Deployment>`.

Since we currently rely on Azure to provide us with a public IP and manage the
DNS entries of MongoDB instances, we detail only the steps required for ACS
here.

Select the current Azure resource group and look for the ``Public IP``
resource. You should see at least 2 entries there - one for the Kubernetes
master and the other for the MongoDB instance.

Select the ``Public IP`` resource that is attached to your service (it should
have the Kubernetes cluster name alongwith a random string),
select ``Configuration`` and add the DNS name that you added in the
ConfigMap earlier.


This will ensure that when you scale the replica set later, other MongoDB
members in the replica set can reach this instance.


Step 9: Run BigchainDB as a Deployment
--------------------------------------

Get the file ``bigchaindb-dep.yaml`` from GitHub using:

.. code:: bash

   $ wget https://raw.githubusercontent.com/bigchaindb/bigchaindb/master/k8s/bigchaindb/bigchaindb-dep.yaml

Note that we set the ``BIGCHAINDB_DATABASE_HOST`` to ``mdb`` which is the name
of the MongoDB service defined earlier.

We also hardcode the ``BIGCHAINDB_KEYPAIR_PUBLIC``,
``BIGCHAINDB_KEYPAIR_PRIVATE`` and ``BIGCHAINDB_KEYRING`` for now.

As we gain more experience running BigchainDB in testing and production, we
will tweak the ``resources.limits`` values for CPU and memory, and as richer
monitoring and probing becomes available in BigchainDB, we will tweak the
``livenessProbe`` and ``readinessProbe`` parameters.

We also plan to specify scheduling policies for the BigchainDB deployment so
that we ensure that BigchainDB and MongoDB are running in separate nodes, and
build security around the globally exposed port ``9984``.

Create the required Deployment using:

.. code:: bash

   $ kubectl apply -f bigchaindb-dep.yaml

You can check its status using the command ``kubectl get deploy -w``


Step 10: Verify the BigchainDB Node Setup
----------------------------------------

Step 10.1: Testing Externally
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Try to access the ``<dns/ip of your exposed service endpoint>:9984`` on your
browser. You must receive a json output that shows the BigchainDB server
version among other things.

Try to access the ``<dns/ip of your exposed service endpoint>:27017`` on your
browser. You must receive a message from MongoDB stating that it doesn't allow
HTTP connections to the port anymore.


Step 10.2: Testing Internally
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Run a container that provides utilities like ``nslookup``, ``curl`` and ``dig``
on the cluster and query the internal DNS and IP endpoints.

.. code:: bash

   $ kubectl run -it toolbox -- image <docker image to run> --restart=Never --rm

It will drop you to the shell prompt.
Now we can query for the ``mdb`` and ``bdb`` service details.

.. code:: bash

   $ nslookup mdb
   $ dig +noall +answer _mdb-port._tcp.mdb.default.svc.cluster.local SRV
   $ curl -X GET http://mdb:27017
   $ curl -X GET http://bdb:9984

There is a generic image based on alpine:3.5 with the required utilities
hosted at Docker Hub under ``bigchaindb/toolbox``.
The corresponding Dockerfile is `here
<https://github.com/bigchaindb/bigchaindb/k8s/toolbox/Dockerfile>`_.
You can use it as below to get started immediately:

.. code:: bash

   $ kubectl run -it toolbox --image bigchaindb/toolbox --restart=Never --rm

