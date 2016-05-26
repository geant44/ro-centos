#### What is this?
This is a step-by-step how-to on how to setup a read-only Cent-OS 7 cluster. It is aimed at anyone wanting to setup a cluster with the following architecture:
  * A "master" server serving the OS to the other computers, through an [NFS](https://en.wikipedia.org/wiki/Network_File_System) server for the files and [PXE](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) boot for the kernel/initial RAM disk.
  * Any number of clients of said server, who will have a read-only root system.

#### Why would I want that?
There are two big advantages at using the preceding configuration, that can be useful in several situations. Firstly, having a read-only root gives you an increase in security and stability, giving you more peace of mind when exposing computers to the public (such as disk-less stations in a library for example), or to code written by other people if you are running servers. The second advantage is that you only have to maintain/administrate "one" computer as all changes are reflected on all the clients. This makes processes such as updates, installation of software and so on much simpler. This setup also gives you the possibility to run the clients disk-less. Interested? Read on!

#### How can I use this?
<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br /><span xmlns:dct="http://purl.org/dc/terms/" href="http://purl.org/dc/dcmitype/Text" property="dct:title" rel="dct:type">CentOS 7 RO Cluster</span> by <span xmlns:cc="http://creativecommons.org/ns#" property="cc:attributionName">Joachim Desroches</span> is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License</a>.

[CentOS](centos.org) is a trademark of RedHat, Inc.
