# SDN-assisted Adaptive Bit Rate streaming (SABR)
This repo contains all source code for implementing network assisted adaptive bitrate video streaming and evaluating the system in the CloudLab testbed.

## About this system
This system has been tested and evaluated in [CloudLab](https://www.cloudlab.us/), a testbed for researchers to build and test clouds and new architectures.
In order to experiment with CloudLab, you will need an account for access and a profile to instantiate. The profile is specified here as <i>cloudlab_topo.rspec</i>.

The following subsystems are included:

## A. Controller and SABR

### Pre-requisites
This component has been tested on a server that runs Ubuntu 14.04 and has the following dependencies:

* [MongoDB](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/)
* OpenNetMon - included in this directory and requires the [PoX controller](https://github.com/noxrepo/pox)
* [R for Ubuntu](https://cran.r-project.org/bin/linux/ubuntu/README.html)
* [Forecast package in R](https://cran.r-project.org/web/packages/forecast/forecast.pdf)
* [rpy2](https://rpy2.readthedocs.io/en/version_2.8.x/) - This is the python library used to invoke R functions like ARIMA from Python.

### Prepare environment and run
1. You will need to setup 3 tables in a MongoDB database; portmonitor, serv_bandwidth, cachemiss where portmonitor will be used to archive network port statistics on active downstream paths, serv_bandwidth is used to store the processed ARIMA forecasts along with cache status and cachemiss is used to archive the incoming cache misses to be used for content placement for various strategies, respectively. For portmonitor and serv_bandwidth tables, you may need to set the maximum size limit using capped collections as specified [here.](https://docs.mongodb.com/manual/core/capped-collections/) For the cachemiss table, it is mandatory to enable capped collections in order for the caching functionality to work. 
[Note:] It is reccommended to run the following processes inside individual [screen](https://www.gnu.org/software/screen/manual/screen.html) processes
2. Run the controller using the following command:
   
      `./pox.py openflow.of_01 --port=<controller_port> log --file=<log_file>,w opennetmon.startup`
3. Run ARIMA forecast and cache status collection module
   a. Copy arima.py to ext folder inside the PoX directory and issue the following command: 
      
      `./pox.py openflow.of_01 --port=<port_number> log --file=<log_file>,w arima`
4. Run the caching component - For the initial setup of the empty caches, the command, `python cacher.py`, must be run only after the orchestration of the testbed experiments has started.

## B. Orchestration - Run experiments on CloudLab testbed.
1. Setup Switches - Create OVS bridges on each of OVS switches, <i>sw1a</i>, <i>sw2a</i>, <i>sw2b</i>, <i>sw3a</i>, <i>sw3b</i>, <i>sw3c</i>, <i>sw3d</i>, <i>sw4a</i>, <i>sw4b</i>, <i>sw4c</i> and <i>sw4d</i> and connect them to the controller using the following commands as a reference:
   
   a. Create bridge
   
   `sudo ovs-vsctl add-br <bridge_name>`
   
   b. Add ports,i.e., interfaces to be controlled
   
   `sudo ovs-vsctl add-port <bridge_name> <port_name>`
   
   c. Connect bridge to OpenFlow controller
   
   `sudo ovs-vsctl set-controller <bridge_name> tcp:<IP_of_controller>:<port_number>`
   
[Note]: For more information on how to configure and work with OVS switches, go [here](http://docs.openvswitch.org/en/latest/tutorials/).

The script, <i>automate_sabr_clab.py</i>, can be updated to remotely execute the above commands on switches if desired.

2. Setup Server 
    * The following dependencies must be installed:
      * screen apache2 python-pip python-dev build-essential libssl-dev libffi-dev mongodb
      * Python libraries: pymongo scapy scapy_http netifaces
    [Hint]: Run the following [script] (https://github.com/dbhat/cloudlab_SABR/blob/master/server/server.sh) on the server.
    
    * Insert metadata information into the cache MongoDB using [<i>create_mpdinfo.py</i>](https://github.com/dbhat/cloudlab_SABR/blob/master/server/create_mpdinfo.py). This script depends on [<i>mpd_insert.py</i>](https://github.com/dbhat/cloudlab_SABR/blob/master/server/mpd_insert.py) and [<i>config_dash.py</i>](https://github.com/dbhat/cloudlab_SABR/blob/master/server/config_dash.py).
    * Use [<i>http_capture.py</i>](https://github.com/dbhat/cloudlab_SABR/blob/master/server/http_capture.py) to listen to http requests for caching. This script is used to sniff HTTP GET requests and works best inside a screen process by running 
    `python http_capture.py`
3. Setup Clients 
    * The following dependencies must be installed:
      * python-pip python-dev build-essential vim screen
      * Python libraries: urllib3 httplib2 pymongo netifaces requests numpy sortedcontainers 
    * In our evaluation, we modify the player available [here](https://github.com/pari685/AStream.git) to obtain the cache map using a GET request of the following format in a [Pymongo](https://api.mongodb.com/python/current/) client: `table.find({"server_ip": str(index), "client_ip":(str(ni.ifaddresses('eth1')[AF_INET][0]['addr'])[:10])}).sort([("_id", pymongo.DESCENDING)]).limit(1)`; where <i>server_ip</i> is the cache which the client wishes to query and <i>client_ip</i> is the client's own IP address which is required as a filter. The query returns a list of segments and qualities found on the cache along with the available bandwidth obtained from ARIMA processing. 
    
    [Note]: You can issue GET requests with Mongo queries with any preferred player of choice since GET requests to MongoDB are widely supported by a range of languages.
4. The script, <i>automate_sabr_clab.py</i> maybe used to automate experiment runs on CloudLab using remote login capability provided by the Python-based [Paramiko](http://www.paramiko.org/) library. The script mainly does the following:
  
    a. Run client algorithm using AStream
  
    b. Resets MongoDB caches for each run/set of runs

5. To run this script on your machine ensure that:

    a. The following Python libraries are installed: numpy, scipy, paramiko, pymongo
  
    b. Copy your SSH keys locally and provide the login credentials in the <i>automate_sabr_clab.py</i> script.
  
    c. Replace the server and client lists with login information in <i>automate_sabr_clab.py</i>.

## C. Parsing - Collect and Analyze Results
1. A sample BASH script, <i>getmultipleruns_BOLA.sh</i>, is provided to collect the results from the CloudLab client machines. This script retrieves results from 60 clients and saves them in different folders to be parsed. You will need to update it with login information of your CloudLab clients. For every algorithm, replace the default, BOLAO, with the name of the client algorithm.
2. For parsing results and computing QoE metrics, average quality bitrate, number of quality switches, spectrum and rebuffering ratio, <i>matplotlib_clab.py</i>, may be used. The script contains parsing logic for BOLAO and BOLAO with SABR. You will need to replace BOLAO with paths for other algorithm results.
3. Cache hit-rates can be computed using the script, <i>cdf_hitratio_qual.py</i>. The current example contains parsing script for BOLAO for the Quality-based caching case. You will need to replace this with other content placement result folders for the Global and Local caching cases.
4. Total content requests per quality can be obtained using the script, <i>cdf_hitratio_qual.py</i>, for BOLAO and SQUAD for the Quality-based caching case. You will need to replace this with other content placement result folders for the Global and Local caching cases.
## D. MATLAB - plotting scripts
1. The script, <i>caching_CDF_SQUAD.m</i>, is used to plot CDF and CCDF graphs for the 4 QoE metrics, Average Quality Bitrate, Number of quality switches, Spectrum and Rebuffering Ratio for all client algorithms. The same script is modified to generate results for the various content placement algorithms for Baseline, Local, Global and Quality-based caching.
2. The script, <i>stacked_requests.m</i>, is used to create a stacked bar graph for the total number of hits for the 5 quality representations for all content placement strategies.
3. The script, <i>caching_hit_ratio.m</i>, is used to create a bar graph for the hit rate, i.e, (no. of requests served by caches)/(total number of requests) for the 5 quality representations for all content placement strategies.


