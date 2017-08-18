.. _kctlr-ingress-config:

Use the BIG-IP Controller as an Ingress
=======================================

.. include:: /_static/reuse/k8s-version-added-1_1.rst

As described in the `Kubernetes Ingress`_ documentation, the IP address assigned to a `Kubernetes Pod`_ can't be accessed from outside of the cluster network.
You can use the |kctlr-long| acts as a `Kubernetes Ingress controller`_ to:

- expose a `Kubernetes Service`_ to external traffic;
- create BIG-IP `Local Traffic Policies`_ for Kubernetes Services;
- :ref:`use BIG-IP SSL profiles for TLS <ingress-TLS>`; and
- :ref:`use BIG-IP health monitors <add health monitor to ingress>` to check the health of the Ingress.

.. attention::

   An Ingress resource corresponds to one (1) BIG-IP virtual server.
   Allocate a BIG-IP `self IP`_ from the external network to use for the virtual server *before* you add the Ingress resource to Kubernetes.

.. _ingress-quick-start:

Quick Start
-----------

You can add the `F5 virtual server properties`_ to any existing Ingress resource.

.. code-block:: console

   kubectl annotate ingress myIngress virtual-server.f5.com/ip="1.2.3.4"
                                      virtual-server.f5.com/partition="k8s"
                                      ingress.kubernetes.io/ssl-redirect="true"
                                      ingress.kubernetes.io/allow-http="false"
                                      kubernetes.io/ingress.class="f5"
                                      virtual-server.f5.com/balance="round-robin"
                                      virtual-server.f5.com/http-port="80"
                                      virtual-server.f5.com/health='[{"path": "svc1.bar.com/foo", "send": "HTTP GET /health/foo", "interval": 5, "timeout": 10}]'


At minimum, you should define the following properties:

- :code:`virtual-server.f5.com/ip`
- :code:`virtual-server.f5.com/partition`

.. hint::

   The :code:`kubernetes.io/ingress.class` property defaults to "f5", so you don't need to include it in your Ingress resource annotation.
   The |kctlr-long| ignores Ingress resources with any other :code:`ingress.class`.

.. _create k8s ingress:

Create a Kubernetes Ingress Resource
------------------------------------

Create a new `Ingress Resource`_ and annotate it with the desired `F5 virtual server properties`_.

.. _single service:

Single Service
``````````````

.. literalinclude:: /_static/config_examples/f5-k8s-single-ingress.yaml
   :linenos:
   :emphasize-lines: 6-10

:fonticon:`fa fa-download` :download:`f5-k8s-single-ingress.yaml </_static/config_examples/f5-k8s-single-ingress.yaml>`

.. _simple fanout:

Simple Fanout
`````````````

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-fanout.yaml
   :linenos:
   :emphasize-lines: 7-12

:fonticon:`fa fa-download` :download:`f5-k8s-ingress-fanout.yaml </_static/config_examples/f5-k8s-ingress-fanout.yaml>`

.. _name-based virtual hosting:

Name-based virtual hosting
``````````````````````````

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-virtual-hosting.yaml
   :linenos:
   :caption: Specific hosts
   :emphasize-lines: 6-14, 18, 26

:fonticon:`fa fa-download` :download:`f5-k8s-ingress-virtual-hosting.yaml </_static/config_examples/f5-k8s-ingress-virtual-hosting.yaml>`

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-virtual-hosting_all.yaml
   :linenos:
   :caption: All hosts
   :emphasize-lines: 6-14, 17

:fonticon:`fa fa-download` :download:`f5-k8s-ingress-virtual-hosting_all.yaml </_static/config_examples/f5-k8s-ingress-virtual-hosting_all.yaml>`

.. _ingress-TLS:

TLS Ingress
-----------

You can create a TLS Ingress using a BIG-IP SSL profile or a Kubernetes Secret.

.. attention::

   If you specify a :code:`spec.tls` section in your Ingress resource but do not provide any properties,the BIG-IP device uses Local traffic policies to redirect HTTP requests to HTTPS.

Use an existing BIG-IP SSL profile
``````````````````````````````````

Take the steps below to use an existing `BIG-IP SSL profile`_ to secure an Ingress.

#. Specify the desired SSL profile(s) in the :code:`spec.tls` section of the Ingress resource.
#. Add the :code:`ingress.kubernetes.io/ssl-redirect` annotation.
   (**OPTIONAL** The :code:`ssl-redirect` parameter defaults to :code:`"true"`
#. Add the :code:`ingress.kubernetes.io/allow-http` annotation.
   (**OPTIONAL** The :code:`allow-http` parameter defaults to :code:`"false"`).

.. note::

   You can specify one (1) or more SSL profiles in your Ingress resource.

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-tls_bigip.yaml
   :caption: TLS Ingress using a BIG-IP SSL Profile


:fonticon:`fa fa-download` :download:`f5-k8s-ingress-tls.yaml </_static/config_examples/f5-k8s-ingress-tls.yaml>`

Use a TLS cert stored in a Kubernetes Secret
````````````````````````````````````````````

Take the steps below to secure an Ingress using a Kubernetes Secret.

.. important::

   The |kctlr| deletes any custom SSL profiles in its BIG-IP partition that aren't sourced from a Kubernetes Secret.

#. :k8s:`Create a Kubernetes Secret from your TLS certificate and key <kctlr-secrets#Add a TLS certificate and key>`_.

#. Specify the name of the Secret in the :code:`spec.tls` section of the Ingress resource.

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-tls_bigip.yaml
   :caption: TLS Ingress using a Kubernetes Secret


.. _add health monitor to ingress:

Add a Health Monitor to an Ingress
----------------------------------

Use the :code:`virtual-server.f5.com/health` annotation with a JSON blob to create a BIG-IP health monitor for any Kubernetes Ingress.

.. literalinclude:: /_static/config_examples/f5-k8s-ingress-health-monitor.yaml
   :caption: Health Monitor Example
   :linenos:
   :emphasize-lines: 9-22


.. _deploy ingress resource:

Deploy the Ingress
------------------

Use :command:`kubectl create` to upload the Ingress Resource to the Kubernetes API server.

.. code-block:: console

   kubectl create ingress -f <filename>.yaml
   Ingress myIngress created

Verify the creation of BIG-IP objects
-------------------------------------

You can use TMOS or the BIG-IP configuration utility to verify that the |kctlr| created the requested BIG-IP objects for your Ingress.

To verify using the BIG-IP configuration utility:

#. Log in to the configuration utility at the management IP address (for example: :code:`https://10.190.25.225/tmui/login.jsp?`).
#. Select the correct partition from the :guilabel:`Partition` drop-down menu.
#. Go to :menuselection:`Local Traffic --> Virtual Servers` to view all virtual servers, pools, and pool members.
#. Go to :menuselection:`Local Traffic --> Policies` to view any new policies.

See the `TMSH Reference Guide`_ (PDF) for the relevant :command:`tmsh ltm` commands.

Learn More
----------

The |kctlr| supports all types of `Kubernetes Ingress`_.

- Single Service: lets you create a BIG-IP virtual server to proxy traffic for a single Service.

- Simple fanout: uses a BIG-IP device as an edge load balancer to proxy requests to endpoints within the cluster.

- Name-based virtual hosting: lets you match host names and paths to a single IP address allocated to a BIG-IP pool.

  .. tip::

     If you don't specify any hosts or paths, the BIG-IP device will proxy traffic for all hosts/paths for the Service specified under :code:`backend`.

   .. table:: Ingress Resource to BIG-IP mapping

      =======================================   ==================================
      Ingress Type                              Description
      =======================================   ==================================
      :ref:`Single Service`                     Create a BIG-IP virtual server
                                                and server pool for a single
                                                Kubernetes Service.
      ---------------------------------------   ----------------------------------
      :ref:`Simple fanout`                      Create a BIG-IP virtual server
                                                and server pools for a group of
                                                Kubernetes Services
                                                (one server pool per Service).
      ---------------------------------------   ----------------------------------
      :ref:`Name-based virtual hosting`         Create a BIG-IP virtual server
                                                and server pools for
                                                Kubernetes Services.

                                                Creates BIG-IP Local Traffic
                                                Policies to route requests to
                                                specific pools according to host
                                                name and path.
      =======================================   ==================================

- TLS Ingress: lets you secure traffic using https on port 443.


.. seealso::

   See the `Kubernetes Ingress`_ documentation for more information.

.. What's Next?
   ------------
   -- add links
   -- to new Ingress use-case
   -- solution docs here

.. _TMSH Reference Guide: https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/bigip-tmsh-reference-12-0-0.html
.. _BIG-IP server pool: https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ltm-basics-12-1-0/4.html
.. _Host header: https://tools.ietf.org/html/rfc7230#section-5.4
.. _Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/#name-based-virtual-hosting
.. _TLS ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/#tls
.. _self IP: https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/tmos-routing-administration-13-0-0/5.html
