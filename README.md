# Splunk-Netflow-Analyzer

#### Description

This project contains all the components and documentation necessary to start collecting and visualizing Netflow data using Splunk. The dashboard allows administrators and security professionals to capture network traffic data, and analyze flows to determine possible bottlenecks and/or security incidents across the corporate network.

##### Key Components:

1. Your Netflow log source (i.e.: Firewall/Network Gateway) - Cisco Meraki was used during this project.
2. Logstash - Version 6.3 was used for this project.
3. Splunk - Version 6.4.3 was used for this project.

##### Flow:

1. The Netflow agent (Firewall) sends data to Logstash on UDP Port 777.
2. Logstash receives the data on UDP Port 777, decodes the Netflow packets.
3. Logstash sends the decoded Netflow packets to Splunk over UDP Port 555.
4. Splunk ingests the packets and displays the visualizations/alerts/etc.

#### Why not use Splunk's built in Netflow collector instead of Logstash?
Splunk has a built-in Netflow collector that can be easily configured using the built-in scripts available as part of the [Splunk Add-on for Netflow](https://splunkbase.splunk.com/app/1658/). More documentation on how to configure the add-on and the binaries to collect Netflow data is available [here](https://docs.splunk.com/Documentation/AddOns/released/NetFlow/Configureinputs).

Unfortunately, the add-on doesn't work, or doesn't work well (from my testing) in a Docker container, which for my purposes Docker seemed to do the job well. I didn't spend much time looking into the issue, but I'm guessing the Netflow binaries need some low level hook access to the OS which might not be available in a Docker container. In a production environment, I would recommend a dedicated server for Splunk depending on how much data you're ingesting, and definitely not in a Docker container or you'll see some pretty big performance impact.

Logstash was simple enought to setup in a separate container, and specifically for this project all I needed was a UDP input, then to decode the data (Netflow codec), then forward the decoded data over to Splunk for Analysis.

# Diagram

#### 

![alt text](https://raw.githubusercontent.com/danucalovj/Splunk-Netflow-Analyzer/master/Netflow-Diagram.png "Diagram")

# Screenshots

#### Geo Map of Flows

![alt text](https://raw.githubusercontent.com/danucalovj/Splunk-Netflow-Analyzer/master/Dashboard-Sample1.PNG "Dashboard Sample 1")

#### Flows by Port, Destination, Etc.

![alt text](https://raw.githubusercontent.com/danucalovj/Splunk-Netflow-Analyzer/master/Dashboard-Sample2.PNG "Dashboard Sample 2")

#### Non-US Traffic Flows, Source and Destination

![alt text](https://raw.githubusercontent.com/danucalovj/Splunk-Netflow-Analyzer/master/Dashboard-Sample3.PNG "Dashboard Sample 3")

# Requirements

##### Software / Network
1. Ubuntu Server 16.04 Xenial
2. Logstash >= v6.3
3. Splunk >= v6.4.3
4. Outbound Firewall Rule: Corp Network > UDP 777 > Ubuntu Server

##### Note: 
This is not a requirement, but if Logstash and Splunk are residing in separate servers on separate networks, Logstash must be able to communicate to UDP Port 555 on the Splunk server. This deployment assumes both reside on the same server.

# Configuration

### Install Logstash

Official Documentation: [Installing Logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)

#### Check Java Version

``` bash
java -version
```

This should produce an output similar to:
``` bash
java version "1.8.0_65"
Java(TM) SE Runtime Environment (build 1.8.0_65-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.65-b01, mixed mode)
```

If Java is not installed, install Java:
``` bash
apt-get install default-jdk
```

#### Install Logstash:
``` bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
sudo apt-get install apt-transport-https
echo "deb https://artifacts.elastic.co/packages/6.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-6.x.list
sudo apt-get update && sudo apt-get install logstash
```

Logstash should be installed now. To verify, navigate to the following folder and verify the binaries are there:
``` bash
/usr/share/logstash/bin
```

#### Install Logstash Plugins:

Run the following commands on the Logstash server:
``` bash
cd /usr/share/logstash/bin
./logstash-plugin install logstash-input-udp
./logstash-plugin install logstash-codec-netflow
./logstash-plugin install logstash-output-udp
```
This should take a few moments for each plugin.

#### Logstash Configuration















