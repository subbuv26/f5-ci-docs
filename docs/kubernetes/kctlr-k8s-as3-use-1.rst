:product: Container Ingress Services :type: concept

.. _kctlr-k8s-as3-use-1:

Container Ingress Services and AS3 Extensions - HTTP application use case
=========================================================================

This use case demonstrates how you can use Container Ingress Services (CIS), and Application Services 3 (AS3) Extenstions to:

- Deploy a simple HTTP application (Container). 
- Expose the application using a Kubernetes Service.
- Configure the BIG-IP system to load balance across the application (PODs).

.. rubric:: **HTTP application overview**

.. image:: /_static/media/cis_http_as3_service.png
   :scale: 70%
           
.. _kctlr-as3-http-use-pre:

Prerequisites
=============

To complete this use case, ensure you have:

- A functioning Kubernetes cluster.
- A BIG-IP system running software version 12.1.x or higher.
- AS3 Extension version 3.10 or higher installed on BIG-IP.
- A BIG-IP system user account with the Administrator role.

.. important::
   If your BIG-IP system is using a self-signed SSL device certificate (the default configuration), include the `--insecure=true` option in your :code:`k8s-bigip-ctlr` deployment. Also, to allow the BIG-IP system to reach containers directly, set the :code:`--pool-member-type=` option to :code:`cluster`.  Your :code:`k8s-bigip-ctlr` deployment should resemble:

.. code-block:: YAML

   args: [
      "--bigip-username=$(BIGIP_USERNAME)",
      "--bigip-password=$(BIGIP_PASSWORD)",
      "--bigip-url=10.10.10.100",
      "--bigip-partition=AS3",
      "--namespace=default",
      "--pool-member-type=cluster",
      "--flannel-name=fl-vxlan",
      "--insecure=true"
         ]

.. _kctlr-as3-http-use-steps:

Procedures
==========

.. _kctlr-as3-http-use-deploy:

I. Deploy the HTTP application 
``````````````````````````````
Kubernetes Deployments are used to create Kubernetes PODs, or applications distributed across multiple hosts. The following Deployment example creates a new application named :code:`f5-hello-world-web`, using the f5-hello-world Docker Container, and the :code:`f5-hello-world-web` label to identify the application. 

.. note::

   Labels are simple key value pairs used to group a set of configuration objects.
   
.. code-block:: YAML

   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: f5-hello-world-web
     namespace: default
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: f5-hello-world-web
     template:
       metadata:
         labels:
           app: f5-hello-world-web
       spec:
         containers:
         - env:
           - name: service_name
             value: f5-hello-world-web
             image: f5devcentral/f5-hello-world:latest
           imagePullPolicy: Always
           name: f5-hello-world-web
           ports:
           - containerPort: 8080
             protocol: TCP

- :fonticon:`fa fa-download` :download:`f5-hello-world-web-deployment.yaml </kubernetes/config_examples/f5-hello-world-web-deployment.yaml>`

To create the Deployment, run the following command on the Kubernetes Master Node: 

.. parsed-literal::

   kubectl apply -f f5-hello-world-service.yaml 

To verify the application is running on the PODs, run: 

.. parsed-literal::

    kubectl get pods | grep f5-hello

    f5-hello-world-web-b48bd87d9-rj9fq            1/1     Running   0          70s
    f5-hello-world-web-b48bd87d9-v867b            1/1     Running   0          70s

.. _kctlr-as3-http-use-expose:

II. Expose the application
``````````````````````````
Kubernetes Services expose applications to external clients. This Service example creates a new Kubernetes Service named :code:`f5-hello-world-web`, and uses labels to identify the application as :code:`f5-hello-world-web`, the Tenent (BIG-IP partition) as :code:`AS3,` and the BIG-IP pool as :code:`web_pool`:

.. note::

   CIS creates BIG-IP pool members using the information in the Kubernetes Service :code:`Endpoints` field. You can view all of the Service fields by running the :code:`kubectl describe services` command.

.. code-block:: YAML

   apiVersion: v1
   kind: Service
   metadata:
     name: f5-hello-world-web
      namespace: default 
      labels:
       app: f5-hello-world-web
       cis.f5.com/as3-tenant: AS3
       cis.f5.com/as3-app: A1
       cis.f5.com/as3-pool: web_pool
   spec:
     ports:
     - name: f5-hello-world-web
       port: 8080
       protocol: TCP
       targetPort: 8080
     type: NodePort
     selector:
       app: f5-hello-world-web

- :fonticon:`fa fa-download` :download:`f5-hello-world-web-service.yaml </kubernetes/config_examples/f5-hello-world-web-service.yaml>`

To create the Kubernetes Service, run the following command on the Kubernetes Master Node:

.. parsed-literal::

   kubectl apply -f f5-hello-world-web-service.yaml 

To verify the Service, run:

.. parsed-literal::

   kubectl describe services f5-hello-world-web 

   Name:                     f5-hello-world-web
   Namespace:                default 
   Labels:                   app=f5-hello-world-web
                             cis.f5.com/as3-app=A1
                             cis.f5.com/as3-pool=web_pool
                             cis.f5.com/as3-tenant=AS3
   Selector:                 app=f5-hello-world-web
   Type:                     NodePort
   IP:                       10.105.126.114
   Port:                     f5-hello-world-web  8080/TCP
   TargetPort:               8080/TCP
   NodePort:                 f5-hello-world-web  32225/TCP
   Endpoints:                10.244.1.121:8080,10.244.2.38:8080
   Session Affinity:         None
   External Traffic Policy:  Cluster

.. _kctlr-as3-http-use-bigip:

III. Configure the BIG-IP system
````````````````````````````````
AS3 ConfigMaps create the BIG-IP system configuration used to load balance across the PODs. This AS3 ConfigMap example creates a ConfigMap named :code:`f5-as3-declaration`. CIS uses this AS3 ConfigMap to create a virtual server, and use Service Discovery to create a load balancing pool named :code:`web_pool` of Service endpoints. The new configuration is created in the AS3 Tenant (BIG-IP partition) :code:`AS3`.

.. code-block:: YAML

   kind: ConfigMap
   apiVersion: v1
   metadata:
     name: f5-as3-declaration
     namespace: default
     labels:
       f5type: virtual-server
       as3: "true"
   data:
     template: |
       {
           "class": "AS3",
           "declaration": {
               "class": "ADC",
               "schemaVersion": "3.10.0",
               "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
               "label": "http",
               "remark": "A1 example",
               "AS3": {
                   "class": "Tenant",
                   "A1": {
                       "class": "Application",
                       "template": "http",
                       "serviceMain": {
                           "class": "Service_HTTP",
                           "virtualAddresses": [
                               "10.192.75.101"
                           ],
                           "pool": "web_pool"
                       },
                       "web_pool": {
                           "class": "Pool",
                           "monitors": [
                               "http"
                           ],
                           "members": [
                               {
                                   "servicePort": 8080,
                                   "serverAddresses": []
                               }
                           ]
                       }
                   }
               }
           }
       }

- :fonticon:`fa fa-download` :download:`f5-hello-world-as3-configmap.yaml </kubernetes/config_examples/f5-hello-world-as3-configmap.yaml>`

To deploy the ConfigMap, run the following command on the Kubernetes Master Node:

.. parsed-literal::

   kubectl create -f f5-hello-world-as3-configmap.yaml

To verify the BIG-IP system has been configured, run: 

.. note::

   Modify the :code:`admin` password, and :code:`https://10.10.10.100` for your BIG-IP system.

.. parsed-literal::

   curl -sk -u admin:admin https://10.10.10.100//mgmt/tm/ltm/virtual/~AS3~A1~serviceMain
   curl -sk -u admin:admin https://10.10.10.100/mgmt/tm/ltm/pool/~AS3~A1~web_pool

.. _kctlr-as3-http-use-delete:

Deleting CIS ConfigMaps
=======================

Because CIS and AS3 use a Declarative API, the BIG-IP system configuration is not removed after you delete a configmap. To remove the BIG-IP system configuration objects created by an AS3 declaration, you must deploy a blank configmap, and restart the controller. Refer to `Deleting CIS AS3 configmaps <kctlr-as3-delete-configmap.html>`_.

You can use this blank ConfigMap to delete the use case ConfigMap and configuration from the BIG-IP system: 

- :fonticon:`fa fa-download` :download:`f5-delete-hello-world-as3-configmap.yaml </kubernetes/config_examples/f5-delete-hello-world-as3-configmap.yaml>`

.. _kctlr-as3-http-use-resource:

Additional AS3 Resources
========================

- `F5 AS3 User Guide`_.
- `F5 AS3 Reference Guide`_.
- `F5 AS3 Installation`_.
- `F5 CIS and AS3 integration  <kctlr-k8s-as3-int.html>`_.

