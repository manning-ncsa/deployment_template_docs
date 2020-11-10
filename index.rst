.. Kubernetes Deployment Framework documentation master file, created by
   sphinx-quickstart on Wed Nov  4 12:34:49 2020.
   You can adapt this file completely to your liking, but it should at least
   contain the root ``toctree`` directive.

Kubernetes Deployment Framework
===========================================================

.. warning::
    This document is currently an incomplete draft.

.. toctree::
   :maxdepth: 2
   :caption: Contents:

   index

Overview
-----------------------------------------------------------

This template repository contains the framework that allows you to deploy services on development and production Kubernetes (k8s) clusters, following the `GitOps <https://www.gitops.tech>`_ philosophy to ensure reproducible cluster states by using version-controlled deployment.

The fundamental unit of our organization scheme is an individual application, or *app*. The idea is that k8s resources (Ingresses, Deployments, Secrets, Services, etc) are typically deployed to support a specific app. We want the creation, update and deletion of these app resources to be as simple as possible. To that end, we are using the `Carvel Kubernetes tools <https://carvel.dev/>`_ as the basis of our workflow.

The repo is organized into several folder trees that follow a specific hierarchy:

* ``/apps/``
* ``/config/``
* ``/infrastructure/``
* ``/private/``
* ``/scripts/``

The foundational configuration of the cluster, including things like namespace definitions and TLS certificates, is essentially app-independent and must be deployed manually from the config files in ``/infrastructure``.

The configuration of individual apps is constructed hierarchically, starting with the parameter values defined in the ``/config`` tree. 

#. The global default values are set first, followed by 
#. cluster-specific, cluster-wide values, followed by 
#. cluster-specific, namespace-specific values, followed by 
#. the values specified in the app-specific folder in the ``/apps`` tree.

.. _carvel:

Carvel Kubernetes Tools
-----------------------------------------------------------

The software utilities powering this framework are part of the Carvel suite (formerly Kubernetes Tools, or k14s). According to the project page:

    Carvel provides a set of reliable, single-purpose, composable tools that aid in your application building, configuration, and deployment to Kubernetes.

These tools include ``ytt`` for configuration templating, ``kbld`` for image building and pushing, and ``kapp`` for creating and updating k8s resources defined together as an application. These tools assume ``kubectl`` is installed.

.. warning::

    Sensitive information, including Kubernetes Secret resources, should not be stored directly in this deployment repo, but should instead be stored in a separate, more secure repo that is cloned in the ``/private`` location. Resources defined in this private git submodule are symbolically linked where needed. 

Infrastructure
-----------------------------------------------------------

All updates to cluster configuration or infrastructure k8s resources should be captured in the ``/infrastructure`` tree. When necessary, scripts to ensure reproducibility of the cluster state should be added to the ``/scripts`` folder **along with the corresponding documentation**.

For example, the image registry credentials needed by the deployments is stored in ``/private/infrastructure/common/registry-auth.yaml`` and symlinked to ``/infrastructure/common/registry-auth.yaml``. It must be deployed manually to each namespace where there is an app deployed that relies on those credentials.

App Deployment
-----------------------------------------------------------

App deployment is best explained with an example. Let's define and deploy an app called ``website``. 

First we create the app folder ``/apps/website``. The next level of the app folder hierarchy must have at least the folder ``common``, and may also contain ``cluster-dev`` and ``cluster-prod``, if there are cluster-specific configuration settings for the app::

    /apps/website/
    /apps/website/common/
    /apps/website/cluster-dev/
    /apps/website/cluster-prod/

The ``common`` folder must contain a ``config`` folder that defines the k8s resources to be created. Any YAML file in this folder will be included as part of the deployed app. This folder typically contains at least two files::

    /apps/website/common/config/config.yaml
    /apps/website/common/config/values.yaml

but also frequently includes a symlink to a `secrets.yaml` file residing in the separate ``/private`` git submodule::

    /apps/website/common/config/secrets.yaml -> ../../../../private/apps/website/common/config/secrets.yaml

The ``values.yaml`` file is a special file used by the ``ytt`` utility to render the templated configuration files. It may contain definitions such as ::

    #@data/values
    ---
    app_name: website

that allow the k8s resource definitions in ``config.yaml`` such as ::

    #@ load("@ytt:data", "data")
    #@yaml/text-templated-strings
    ---
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: (@= data.values.app_name @)

to render as ::

    ---
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: website

If we want to actually deploy this to the cluster, we invoke the ``deploy`` script::

    /scripts/deploy -a website -n default -c dev -u cluster-user

The input parameters tell the deploy script to build and deploy the app "website" to the namespace "default" on the development cluster with authentication for user "cluster-user". Let's follow the logic of the ``deploy`` script to understand the hierarchical configuration scheme.

First, the k8s access configuration file, or kubeconfig, for the target cluster is loaded for use by ``kubectl``::

    export KUBECONFIG=/private/infrastructure/cluster-dev/.kube/config

Then the context is set for subsequent ``kubectl`` commands::

    kubectl config use-context cluster-user@cluster-dev

The next step is the hierarchical loading of the app configuration. The order follows this sequence::

    1. /config/defaults
    2. /config/cluster-dev/cluster
    3. /config/cluster-dev/namespace
    4. /apps/website/common/config
    5. /apps/website/cluster-dev/config

The power of this hierarchy is evident when we dive into more detail in this example.

Development workflow
-----------------------------------------------------------

The primary deployment workflow revolves around the ``/scripts/deploy`` script, deploying an updated app first on the dev cluster, and then after code review, on the production cluster.

#. A developer clones this repo (recursively to get the submodules), and then typically begins working on a particular app's source code (typically a git submodule within ``/apps/[app_name]/common/build/src``).
#. To rapidly iterate the code, the developer frequently runs the deploy script to update the resources on the dev cluster in a development namespace such as ``dev1``. The ``deploy`` script builds the templated k8s resource config files for the app using ``ytt``, builds the updated Docker image if necessary using ``kbld``, pushes the image to the image repository, and then updates the app resources using ``kapp``.
#. When the app update has been tested properly, it can be deployed on the production cluster by simply changing the target cluster in the ``deploy`` command.

Documentation
-----------------------------------------------------------

The documentation is powered by Sphinx, using the reStructuredText (reST) format. With Sphinx installed locally, you can build the HTML files using ::

  cd docs/
  make html

Then open ``docs/_build/html/index.html`` in your browser.
