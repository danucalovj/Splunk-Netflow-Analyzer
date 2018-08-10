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

### Install Logstash:

Official Documentation: [Installing Logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)

#### Check Java Version:

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

#### Logstash Configuration:

Now that the plugins are installed, we need to load the configuration that tells Logstash to listen for packets on UDP port 777, decode the Netflow packets, and then send them off to Splunk on UDP port 555.

``` bash
cd /etc/logstash/conf.d
touch netflow.conf
nano netflow.conf
```
Copy and paste the following into the netflow.conf file:
``` yml
input {
  udp {
    port => 777
    codec => netflow {
      versions => [5, 9]
    }
    type => netflow
    tags => "port_777"
  }
}

output {
  stdout { }
  udp {
    host => "127.0.0.1"
    port => 555
  }
}
```

##### Note:
If using a separate server for Splunk, replace "127.0.0.1" with the IP of your Splunk server.

### Configure Splunk:

Now we need to configure a few settings on the Splunk server...

#### Install the Splunk Technical Add-on for Netflow:

[Splunk Add-on for Netflow](https://splunkbase.splunk.com/app/1658/)

This will load a few things on your Splunk server, including a new "netflow" source type that automatically parses netflow data and maps specific related fields.

#### Install the App (Optional - Or use the dashboard XML below this section)

I've created an app for Splunk that can be easily installed on the server with a few dashboards for visualizing the incoming Netflow data. To install on the (Ubuntu) Splunk server, do the following:

``` bash
cd /opt/splunk/etc/apps
wget https://github.com/danucalovj/Splunk-Netflow-Analyzer/raw/master/netflow_analyzer.tar.gz
tar -xvf netflow_analyzer.tar.gz
rm netflow_analyzer.tar.gz
```
Restart the Splunk server:

Splunk > Settings > Server Controls > Restart Splunk

Once you login again to Splunk, the App should be loaded on the left in the main Launcher screen.

#### Dashboard XML

If you prefer to load the dashboard manually, for example, if you're ingesting the Netflow data within the context of another app (next section), do the following:

Splunk > Preferred (or Default Search App) > Dashboards > Create New Dashboard

Set a title for your new dashboard, and the appropriate description, id, permissions, etc.

On the new dashboard > Edit Source > Copy and Paste the following XML ([Also available here](https://github.com/danucalovj/Splunk-Netflow-Analyzer/raw/master/traffic_analysis.xml)) and save :

``` XML
<dashboard>
  <label>Traffic Analysis</label>
  <row>
    <panel>
      <map>
        <search>
          <query>source="netflow" | iplocation netflow.ipv4_dst_addr | geostats count(Country) latfield=lat longfield=lon</query>
          <earliest>-24h@h</earliest>
          <latest>now</latest>
        </search>
        <option name="mapping.choroplethLayer.colorBins">5</option>
        <option name="mapping.choroplethLayer.colorMode">auto</option>
        <option name="mapping.choroplethLayer.maximumColor">0xDB5800</option>
        <option name="mapping.choroplethLayer.minimumColor">0x2F25BA</option>
        <option name="mapping.choroplethLayer.neutralPoint">0</option>
        <option name="mapping.choroplethLayer.shapeOpacity">0.75</option>
        <option name="mapping.choroplethLayer.showBorder">1</option>
        <option name="mapping.data.maxClusters">100</option>
        <option name="mapping.drilldown">all</option>
        <option name="mapping.map.center">(0,0)</option>
        <option name="mapping.map.panning">1</option>
        <option name="mapping.map.scrollZoom">0</option>
        <option name="mapping.map.zoom">2</option>
        <option name="mapping.markerLayer.markerMaxSize">50</option>
        <option name="mapping.markerLayer.markerMinSize">10</option>
        <option name="mapping.markerLayer.markerOpacity">0.8</option>
        <option name="mapping.showTiles">1</option>
        <option name="mapping.tileLayer.maxZoom">7</option>
        <option name="mapping.tileLayer.minZoom">0</option>
        <option name="mapping.tileLayer.tileOpacity">1</option>
        <option name="mapping.type">marker</option>
      </map>
    </panel>
  </row>
  <row>
    <panel>
      <chart>
        <title>In Bytes</title>
        <search>
          <query>source="netflow" sourcetype="netflow"| bucket _time span=1m | stats sum("netflow.in_bytes") as "In Bytes" by _time</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">area</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
    <panel>
      <chart>
        <title>Out Bytes</title>
        <search>
          <query>source="netflow" sourcetype="netflow"| bucket _time span=1m | stats sum("netflow.out_bytes") as "Out Bytes" by _time</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">area</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
  </row>
  <row>
    <panel>
      <chart>
        <title>Top Flows by Destination Port</title>
        <search>
          <query>source="netflow"| where "netflow.l4_dst_port" != "0"| stats count("netflow.l4_dst_port") by "netflow.l4_dst_port" | sort count("netflow.l4_dst_port") desc | head 10</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">pie</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
    <panel>
      <chart>
        <title>Top Flows by Destination Port Over Time</title>
        <search>
          <query>source="netflow"| bucket _time span=30s| chart count("netflow.l4_dst_port") by _time,"netflow.l4_dst_port"</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">area</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
  </row>
  <row>
    <panel>
      <chart>
        <title>Top Destination IPs</title>
        <search>
          <query>source="netflow" | stats count(netflow.ipv4_dst_addr) as "Hits" by netflow.ipv4_dst_addr | sort "Hits" desc | head 20</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">pie</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
    <panel>
      <chart>
        <title>Top Destination Countries</title>
        <search>
          <query>source="netflow" | iplocation netflow.ipv4_dst_addr | stats count(Country) as "Hits" by Country | sort "Hits" desc</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">pie</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
  </row>
  <row>
    <panel>
      <chart>
        <title>Non-US Traffic Flows by Country (4h)</title>
        <search>
          <query>source="netflow" | iplocation netflow.ipv4_dst_addr | where Country != "United States" | rename netflow.ipv4_src_addr as "Source IP" | rename netflow.ipv4_dst_addr as "Destination IP" | rename netflow.l4_dst_port as "Destination Port" | table _time Country "Source IP" "Destination IP" "Destination Port" netflow.out_bytes | bucket _time span=1m | chart count(netflow.out_bytes) by _time,Country</query>
          <earliest>-4h@m</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.axisLabelsX.majorLabelStyle.overflowMode">ellipsisNone</option>
        <option name="charting.axisLabelsX.majorLabelStyle.rotation">0</option>
        <option name="charting.axisTitleX.visibility">visible</option>
        <option name="charting.axisTitleY.visibility">visible</option>
        <option name="charting.axisTitleY2.visibility">visible</option>
        <option name="charting.axisX.scale">linear</option>
        <option name="charting.axisY.scale">linear</option>
        <option name="charting.axisY2.enabled">0</option>
        <option name="charting.axisY2.scale">inherit</option>
        <option name="charting.chart">area</option>
        <option name="charting.chart.bubbleMaximumSize">50</option>
        <option name="charting.chart.bubbleMinimumSize">10</option>
        <option name="charting.chart.bubbleSizeBy">area</option>
        <option name="charting.chart.nullValueMode">gaps</option>
        <option name="charting.chart.showDataLabels">none</option>
        <option name="charting.chart.sliceCollapsingThreshold">0.01</option>
        <option name="charting.chart.stackMode">default</option>
        <option name="charting.chart.style">shiny</option>
        <option name="charting.drilldown">all</option>
        <option name="charting.layout.splitSeries">0</option>
        <option name="charting.layout.splitSeries.allowIndependentYRanges">0</option>
        <option name="charting.legend.labelStyle.overflowMode">ellipsisMiddle</option>
        <option name="charting.legend.placement">right</option>
      </chart>
    </panel>
  </row>
  <row>
    <panel>
      <table>
        <title>Live Traffic by Non-US Destinations (60m)</title>
        <search>
          <query>source="netflow" | iplocation netflow.ipv4_dst_addr | where Country != "United States" | rename netflow.ipv4_src_addr as "Source IP" | rename netflow.ipv4_dst_addr as "Destination IP" | rename netflow.l4_dst_port as "Destination Port" | table _time Country "Source IP" "Destination IP" "Destination Port"</query>
          <earliest>-60m@m</earliest>
          <latest>now</latest>
        </search>
      </table>
    </panel>
  </row>
</dashboard>
```

#### Create the UDP input to receive Netflow data from Logstash:

Splunk > Settings > Data inputs > UDP > New

Port: 555
Source Type: netflow
App Context: Select the app context you want use for this input, or leave as default.

Review and finish the wizard.

#### Start Logstash

Now it's time to start the logstash server:

``` bash
service logstash restart
```

Or, if the service is already stopped:
``` bash
service logstash start
```

Alternatively, you can manually run the configuration to test it:
``` bash
/usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/netflow.conf
```

# Conclusion
That's it! Now you should have a deployment capable of ingesting, parsing and visualizing your Netflow data. Simply configure your firewall to send Netflow to the Logstash server on port 777 and your data should start showing within a few minutes.

## What's Next?
Over coming weeks/months, I'll be adding a significant amount of functionality to this dashboard. Right now, it's pretty simple, but the following changes/commits will happen in the near future:

1. Threat Intelligence - Correlating incoming Netflow data to Firehol IPSETS to identify possible signs of compromise, outgoing TOR traffic, etc.
2. Reports
3. Alerts
4. Integration with 3rd party tools/services. i.e.: Alerts on specific events via Zapier, webhooks, Twilio, etc.
5. Integration with Cloudwatch for log storage.













