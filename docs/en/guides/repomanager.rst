=============================================
[Linux] - Repomanager, repository manager
=============================================

.. raw:: html

    <div align="center">
        <a href="https://raw.githubusercontent.com/lbr38/repomanager/refs/heads/devel/images/readme/github-readme-black.png">
        <img src="https://raw.githubusercontent.com/lbr38/repomanager/refs/heads/devel/images/readme/github-readme-black.png" align="top"> 
        </a>
    </div> 

    <br>

**Repomanager** is an open-source web interface to create, maintain, and operate mirrors of **deb** and **rpm** package repositories.

The project is available on GitHub: https://github.com/lbr38/repomanager

Overview
--------

In a server fleet, managing updates consistently can quickly become complex. Repomanager addresses this need by centralizing:

- creation of mirror repositories (internal or synchronized from external sources),
- controlled publication of repository versions,
- monitoring of client machine status,
- execution of scheduled tasks around repositories.

The goal is to make deployments more reliable and to better control what is installed across environments (test, preprod, prod, etc.).

.. raw:: html

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/demo.gif">
        <img src="https://assets.repomanager.net/repomanager/index/demo.gif" align="top"> 
        </a>
    </div> 

    <br>

Key Features
------------

- **deb/rpm repository management**: create local or mirror repositories, duplicate, delete, and rebuild metadata.
- **Environment-based publication**: ability to associate a repository snapshot with an environment (for example preprod, then prod).
- **Package upload**: add custom packages to a local repository through the web interface.
- **GPG signing**: sign packages and repositories to strengthen integrity and trust on the client side.
- **Task scheduling**: immediate or delayed execution of tasks (mirror update, rebuild, publication, etc.).
- **Host tracking**: inventory of installed packages, packages to update, and operations history.
- **Host profiles**: define configuration profiles (repository access, package exclusions) to standardize the fleet.
- **Statistics and monitoring**: visualization of repository access metrics and overall activity.

.. raw:: html

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-1.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-1.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-2.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-2.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-3.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-3.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-4.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-4.png" align="top"> 
        </a>
    </div> 

    <br>

    <div align="center">
        <a href="https://assets.repomanager.net/repomanager/index/index-5.png">
        <img src="https://assets.repomanager.net/repomanager/index/index-5.png" align="top"> 
        </a>
    </div> 

    <br>


Typical Use Case
----------------

A common workflow is:

1. Create a mirror of a distribution (Debian, Rocky Linux, etc.).
2. Validate packages in a test environment.
3. Point the production environment to the validated snapshot.
4. Trigger or schedule updates for the relevant hosts.

This approach limits side effects and makes rollbacks easier in case of incidents.

Installation
------------

For hardware/software prerequisites and installation steps, refer to the official documentation:

- Requirements: https://docs.repomanager.net/getting-started/requirements/
- Standard installation: https://docs.repomanager.net/getting-started/installation/

The official documentation also covers environment configuration, source repositories, reverse proxy setup, and advanced administration.

Further Reading
---------------

- Full documentation: https://docs.repomanager.net/
- GitHub repository: https://github.com/lbr38/repomanager
- Issues and feature requests: https://github.com/lbr38/repomanager/issues
