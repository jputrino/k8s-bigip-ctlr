F5 BIG-IP Controller for Kubernetes
===================================

.. toctree::
    :hidden:
    :maxdepth: 2

    RELEASE-NOTES
    /_static/ATTRIBUTIONS

The |kctlr-long| manages F5 BIG-IP `Local Traffic Manager <https://f5.com/products/big-ip/local-traffic-manager-ltm>`_ (LTM) objects from `Kubernetes`_.

|release-notes|

|attributions|

:fonticon:`fa fa-download` :download:`Attributions.md </_static/ATTRIBUTIONS.md>`

Features
--------
- Dynamically creates, manages, and destroys BIG-IP objects.
- Forwards traffic from BIG-IP to `Kubernetes clusters`_ via `NodePorts`_ or `ClusterIPs`_.
- Support for F5 `iApps`_.

Guides
------

See the `F5 Container Connector for Kubernetes user documentation </containers/v1/kubernetes/>`_.

Overview
--------

The |kctlr-long| is a Docker container that runs in a `Kubernetes`_ Pod.
It uses an F5 Resource to determine:

- what objects to configure on your BIG-IP, and
- to which `Kubernetes Service`_ the BIG-IP objects belong.

The |kctlr-long| watches the Kubernetes API for the creation and modification of F5 resources.
When it discovers changes, the |kctlr-long| modifies the BIG-IP accordingly.


For example:

#. |kctlr-long| discovers a new F5 ``virtualServer`` resource.
#. |kctlr-long| creates a new virtual server object on the BIG-IP. [#objectpartition]_
#. |kctlr-long| creates a pool member on the virtual server for each node in the cluster. [#nodeport]_
#. |kctlr-long| monitors F5 resources, and linked Kubernetes resources, for changes.
#. |kctlr-long| reconfigures the BIG-IP when it discovers changes.

The BIG-IP handles traffic for the Service the specified virtual address and load-balances to all nodes in the cluster. Within the cluster, the allocated NodePort load balances traffic to all pods.

Configuration Parameters
------------------------

+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| Parameter          | Type    | Required | Default     | Description                             | Allowed Values |
+====================+=========+==========+=============+=========================================+================+
| bigip-username     | string  | Required | n/a         | BIG-IP iControl REST username           |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| bigip-password     | string  | Required | n/a         | BIG-IP iControl REST password           |                |
|                    |         |          |             | [#secrets]_                             |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| bigip-url          | string  | Required | n/a         | BIG-IP admin IP address                 |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| bigip-partition    | string  | Required | n/a         | The BIG-IP partition in which           |                |
|                    |         |          |             | to configure objects.                   |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| namespace          | string  | Optional | All         | Kubernetes namespace(s) to watch, if not|                |
|                    |         |          |             | provided will watch all namespaces      |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| namespace-label    | string  | Optional | n/a         | Tells the ``k8s-bigip-ctlr`` to watch   |                |
|                    |         |          |             | any namespace with this label           |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| kubeconfig         | string  | Optional | ./config    | Path to the *kubeconfig* file           |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| python-basedir     | string  | Optional | /app/python | Path to python utilities                |                |
|                    |         |          |             | directory                               |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| running-in-cluster | boolean | Optional | true        | Indicates whether or not a              | true, false    |
|                    |         |          |             | kubernetes cluster started              |                |
|                    |         |          |             | ``k8s-bigip-ctlr``                      |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| use-node-internal  | boolean | Optional | true        | filter Kubernetes InternalIP            | true, false    |
|                    |         |          |             | addresses for pool members              |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| verify-interval    | integer | Optional | 30          | In seconds, interval at which           |                |
|                    |         |          |             | to verify the BIG-IP                    |                |
|                    |         |          |             | configuration.                          |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| node-poll-interval | integer | Optional | 30          | In seconds, interval at which           |                |
|                    |         |          |             | to poll the cluster for its             |                |
|                    |         |          |             | node members.                           |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| log-level          | string  | Optional | INFO        | Log level                               | INFO,          |
|                    |         |          |             |                                         | DEBUG,         |
|                    |         |          |             |                                         | CRITICAL,      |
|                    |         |          |             |                                         | WARNING,       |
|                    |         |          |             |                                         | ERROR          |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| pool-member-type   | string  | Optional | nodeport    | Create this type of BIG-IP pool members | cluster,       |
|                    |         |          |             |                                         | nodeport       |
|                    |         |          |             | Use ``cluster`` to create pool members  |                |
|                    |         |          |             | for each of the endpoints for the       |                |
|                    |         |          |             | service. e.g. the pod's ip              |                |
|                    |         |          |             |                                         |                |
|                    |         |          |             | Use ``nodeport`` to create pool members |                |
|                    |         |          |             | for each schedulable node using the     |                |
|                    |         |          |             | service's NodePort                      |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+
| openshift-sdn-name | string  | Optional | n/a         | BigIP configured VxLAN name             |                |
|                    |         |          |             | for access into the Openshift           |                |
|                    |         |          |             | SDN and Pod network                     |                |
+--------------------+---------+----------+-------------+-----------------------------------------+----------------+


F5 Resource Properties
----------------------

F5 Resources are JSON blobs encoded within Kubernetes ConfigMaps. The ConfigMap must contain the following properties:

+---------------+---------------------------------------------------+-----------------------------------------------+
| Property      | Description                                       | Allowed Values                                |
+===============+===================================================+===============================================+
| f5type        | Defines the type of object                        | virtual-server                                |
|               | ``k8s-bigip-ctlr`` creates on the BIG-IP          |                                               |
+---------------+---------------------------------------------------+-----------------------------------------------+
| schema        | Verifies the ``data`` blob                        | f5schemadb://bigip-virtual-server_v0.1.3.json |
+---------------+---------------------------------------------------+-----------------------------------------------+
| data          | Defines the F5 resource                           |                                               |
+---------------+---------------------------------------------------+-----------------------------------------------+
| frontend      | Defines object(s) created on the BIG-IP           | See `frontend <#frontend>`_                   |
+---------------+---------------------------------------------------+-----------------------------------------------+
| backend       | Identifes the Kubernets Service acting as the     | See `backend <#backend>`_                     |
|               | server pool                                       |                                               |
+---------------+---------------------------------------------------+-----------------------------------------------+

Frontend
````````

virtualServer
~~~~~~~~~~~~~

==================== ================= ============== =========== ===================================================== ======================
Property             Type              Required       Default     Description                                           Allowed Values
==================== ================= ============== =========== ===================================================== ======================
partition            string            Required                   Define the BIG-IP partition to manage

virtualAddress       JSON object       Optional                   Allocate a virtual address from the BIG-IP

- bindAddr           string            Required                   Virtual IP address
- port               integer           Required                   Port number

mode                 string            Optional       tcp         Set the proxy mode                                    http, tcp

balance              string            Optional       round-robin Set the load balancing mode                           round-robin

sslProfile           JSON object       Optional                   BIG-IP SSL profile to apply to the virtual server.

- f5ProfileName      string            Optional                   Name of the BIG-IP SSL profile you want to use.

                                                                  Uses format :code:`partition_name/cert_name`

                                                                  Example: :code:`Common/testcert`

- f5ProfileNames     array of strings  Optional                   Array of BIG-IP SSL profile names.

                                                                  Each SSL profile name uses the format
                                                                  :code:`partition_name/cert_name`.

                                                                  Example: ::

                                                                    [
                                                                      'Common/testcert1',
                                                                      'Common/testcert2'
                                                                    ]
==================== ================= ============== =========== ===================================================== ======================


If ``bindAddr`` is not provided in the Frontend configuration, then you must supply it via a `Kubernetes Annotation`_ for the ConfigMap. The controller watches for the annotation key ``virtual-server.f5.com/ip``.
This annotation must contain the IP address that the virtual server will use. You can configure an IPAM system to write out this annotation containing the IP address that it chose.

A user of the Kubernetes API can check the ``status.virtual-server.f5.com/ip`` annotation, set by the controller, to see the ``bindAddr`` that the virtual server is using.

If ``virtualAddress`` or ``bindAddr`` are not provided in the Frontend configuration, then the controller will configure and manage pools, pool members, and healthchecks for the service without a virtual server on the BIG-IP.
Instead you should already have a BIG-IP virtual server that handles client connections and has an irule or traffic policy to forward the request to the correct pool. The stable name of the pool will be the namespace
of the Kubernetes service followed by an underscore followed by the name of the service ConfigMap.

**To configure multiple SSL profiles**, use ``f5ProfileNames``, not ``f5ProfileName``. ``f5ProfileName`` and ``f5ProfileNames`` are mutually exclusive.

iApps
~~~~~

+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| Property            | Type      | Required  | Default   | Description                                           | Allowed Values            |
+=====================+===========+===========+===========+=======================================================+===========================+
| partition           | string    | Required  |           | Define the BIG-IP partition                           |                           |
|                     |           |           |           | to manage.                                            |                           |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| iapp                | string    | Required  |           | BIG-IP iApp template to use                           | Any iApp template already |
|                     |           |           |           | to create the application                             | configured on the BIG-IP. |
|                     |           |           |           | service.                                              |                           |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| iappPoolMemberTable | JSON      | Required  |           | Define the name and layout of the pool-member table   |                           |
|                     | object    |           |           | in the iApp.                                          |                           |
|                     |           |           |           | See the iApp Pool Member Table section below.         |                           |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| iappTables          | JSON      | Optional  |           | Define iApp tables to apply to                        |                           |
|                     | object    |           |           | the Application Service                               |                           |
|                     | array     |           |           |                                                       |                           |
|                     |           |           |           | Example:                                              |                           |
|                     |           |           |           | ``"iappTables": {``                                   |                           |
|                     |           |           |           | ``"monitor__Monitors":``                              |                           |
|                     |           |           |           | ``{"columns": ["Index", "Name", "Type", "Options"],`` |                           |
|                     |           |           |           | ``"rows": [[0, "mon1", "tcp", "" ],``                 |                           |
|                     |           |           |           | ``[1, "mon2", "http", ""]]}}"``                       |                           |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| iappOptions         | key-value | Required  |           | Define the App configurations                         | See configuration         |
|                     | object    |           |           |                                                       | parameters above.         |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+
| iappVariables       | key-value | Required  |           | Define the iApp variables                             |                           |
|                     | object    |           |           | needed for service creation.                          |                           |
+---------------------+-----------+-----------+-----------+-------------------------------------------------------+---------------------------+

iApp Pool Member Table
``````````````````````

You can use the ``iappPoolMemberTable`` option to describe the layout of the pool-member table that the controller should configure.  It is a JSON object with these properties:

- ``name`` (required): A string that specifies the name of the table that contains the pool members.
- ``columns`` (required): An array that specifies the columns that the controller will configure in the pool-member table, in order.

Each entry in ``columns`` is an object that has a ``name`` property and either a ``kind`` or ``value`` property:

- ``name`` (required): A string that specifies the column name.
- ``kind``: A string that tells the controller what property from the node to substitute.  The controller supports ``"IPAddress"`` and ``"Port"``.
- ``value``: A string that specifies a value.  The controller will not perform any substitution, it uses the value as specified.

For instance, if you configure an application with two pods at 1.2.3.4:20123 and 1.2.3.5:20321, and you specify::

    "iappPoolMemberTable" = {
      "name": "pool__members",
      "columns": [
        {"name": "Port", "kind": "Port"},
        {"name": "IPAddress", "kind": "IPAddress"},
        {"name": "ConnectionLimit", "value": "0"}
      ]
    }

This would configure the following table on BIG-IP::

    {
      "name": "pool__members",
      "columnNames": [
        "Port",
        "IPAddress",
        "ConnectionLimit",
      ],
      "rows": [
        {
          "row": [
            "20123",
            "1.2.3.4",
            "0",
          ]
        },
        {
          "row": [
            "20321",
            "1.2.3.5",
            "0",
          ]
        },
      ]
    }

You will need to adjust this for the particular iApp template that you are using.  One way to discover the format is to configure an iApp manually from a template, and then check its configuration using ``tmsh list sys app service <appname>``.


Backend
```````

+---------------+-----------+-----------+-----------+-------------------------------+---------------------------+
| Property      | Type      | Required  | Default   | Description                   | Allowed Values            |
+===============+===========+===========+===========+===============================+===========================+
| serviceName   | string    | Required  | none      | The `Kubernetes Service`_     |                           |
|               |           |           |           | representing the server pool. |                           |
+---------------+-----------+-----------+-----------+-------------------------------+---------------------------+
| servicePort   | integer   | Required  | none      | Kubernetes Service port       |                           |
|               |           |           |           | number                        |                           |
+---------------+-----------+-----------+-----------+-------------------------------+---------------------------+
| healthMonitors| JSON      | Optional  | none      | Array of TCP or HTTP Health   |                           |
|               | object    |           |           | Monitors.                     |                           |
|               | array     |           |           |                               |                           |
+---------------+-----------+-----------+-----------+-------------------------------+---------------------------+


Example Configuration Files
```````````````````````````

Click to download our sample config files.

- :examples:`sample-k8s-bigip-ctlr-secrets.yaml`
- :examples:`sample-bigip-credentials-secret.yaml`
- :examples:`example-vs-resource.configmap.yaml`
- :examples:`example-vs-resource.json`
- :examples:`example-vs-resource-iapp.json`
- :examples:`example-advanced-vs-resource-iapp.json`

.. rubric:: Footnotes
.. [#objectpartition]  The |kctlr-long| creates and manages objects in the BIG-IP partition defined in the `F5 resource </containers/v1/kubernetes/index.html#f5-resource-properties>`_ ConfigMap.
.. [#nodeport]  The |kctlr-long| forwards traffic to the NodePort assigned to the service by Kubernetes; see the `Kubernetes Services documentation <http://kubernetes.io/docs/user-guide/services/>`_ :fonticon:`fa fa-external` for more information.
.. [#secrets]  You can store sensitive information as a `Kubernetes Secret <http://kubernetes.io/docs/user-guide/secrets/>`_ :fonticon:`fa fa-external`.




.. _Kubernetes: https://kubernetes.io/
.. _Kubernetes Service: https://kubernetes.io/docs/user-guide/services/
.. _Kubernetes Annotation: https://kubernetes.io/docs/user-guide/annotations/
.. _Kubernetes clusters: https://kubernetes.io/docs/admin/
.. _NodePorts: https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport
.. _ClusterIPs: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types
.. _iApps: https://devcentral.f5.com/iapps
.. _Kubernetes pods: https://kubernetes.io/docs/user-guide/pods/
.. _Kubernetes Ingress resources: https://kubernetes.io/docs/user-guide/ingress/
.. _iApp table: https://devcentral.f5.com/wiki/iApp.Working-with-Tables.ashx
.. _Kubernetes Service Type: https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services---service-types
