# Graylog Barnyard2 Log Parsing

Inspired by [devopstales pfsense parser](https://github.com/devopstales/pfsense-graylog).

Using this guide we are able to take logs generated from Snort Barnyard2 (within pfSense) and parse them in Graylog to be able to use the information to pipe into Grafana.

## Prerequisites

1. pfSense with Snort running
2. Graylog (Version 3.2.0+)
3. Grafana (Optional, but recommended, see Grafana section for requirements)
 
If you don't have those 3 running, you'll need to get them setup in your environment before continuing.

Note: *If you're running a separate elasticsearch backend other than what Graylog uses you cannot run the OSS version with this.*

## Graylog Pre-Configuration

I call this the "pre-configuration" because it's what we need to do before we get into the real meat and potatoes of this. Follow the steps below to get Graylog ready to parse logs from Snort within pfSense.

1. Create a new index set with the settings below

	![Image of Barnyard2 index set](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/barnyard2_index_config.PNG)

2. Download the `snort_barnyard2_graylog_content_pack.json` from this repository and go to **System -> Content Packs** click "Upload" in the top right and upload the JSON file.

> This content pack will create the inputs, streams, pipelines, pipeline rules, lookup tables, lookup caches and lookup tables needed to properly parse the needed logs.
3. Go to **Streams** and edit the `Snort Barnyard2 Logs` stream, check `Remove matches from ‘All messages’ stream` and set the index set to the one you just created. We check `Remove matches from ‘All messages’ stream` so that we don't store messages twice and we set the index set so it works with our pipelines.

Awesome, Graylog is now "pre-configured" for what we need to do. Lets move onto the next section.

## GeoLite2 DB Installation

Now that you have the content pack installed to fully utilize it and get IP Geo-location you'll need to download the MaxMind GeoLite2 Database (MMDB format) and place the file on your Graylog server.

1. Go to [MaxMind](https://dev.maxmind.com/geoip/geoip2/geolite2/) and click **Sign Up For GeoLite2** at the bottom. 
2. Create an account with MaxMind and sign in
3. Once you're signed in, click "Download Databases"

	![Screenshot of Maxmind download](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/maxmind_download_databases.PNG)

4. Click **Download GZIP** next to **GeoLite2 City** (DO NOT DOWNLOAD THE CSV FORMAT)

	![Screenshot of GeoLite2 City download](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/maxmind_geolite2_download.PNG)

5. Extract the zip file and place the `GeoLite2-City.mmdb` file in **/etc/graylog/server/** on your graylog server.
6. In Graylog go to **System -> Configurations** and click **Update** under **Geo-Location Processor**
7. Set `/etc/graylog/server/GeoLite2-City.mmdb` as the path and choose **City Database** as the type and click **Save**
8. Scroll to the top and click **Update** under **Message Processors Configuration** and change the order to what is below

	![Screenshot of message processors configuration](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/graylog_message_processors_configuration.PNG)

## Elasticsearch Configuration

Underneath the hood of Graylog runs Elasticsearch. Elasticsearch is what is storing our logs in "indexes". We need to use a tool called [Cerebro](https://github.com/lmenezes/cerebro) to modify our `Barnyard2 Logs` index so that it templates the coordinates properly.

 You'll need to download Cerebro and be able to run it from a Linux box. I personally run this from my Graylog server when needed. So go to the Cerebro link above and `git clone` the repository to your home directory.

Now that we have the repository cloned we're going to go into `cerebro-*/bin` folder and run Cerebro. I launch it using a few custom variables to not allow it to run on its default port of 9000. Use the command below to run Cerebro. 

` ./cerebro -Dhttp.port=9091 -Dhttp.address=X.X.X.X`

Change X.X.X.X to the primary IP of the server you're running Cerebro on. 

Once you have Cerebro running navigate to the web interface in your browser by going to https://X.X.X.X:9091 and target your graylog server IP and port for Elasticsearch (default 9200). `Ex. http://10.1.1.1:9200` and then click "Connect".

Now that you're in Cerebro we need to create an index template. Go to `More -> Index Templates` and on the right-hand side you can Create a New Template. We're going to call this template `Barnyard2-Custom` and then copy and paste the contents of `elasticsearch_custom_template.json` into the Template section. Once you've done that click **Create** at the bottom. 

Now that we've created the template we need to stop the Graylog service by running `systemctl stop graylog-server` on your Graylog server. Once that is stopped we need to delete the `barnyard_0` index visible under **Overview** in Cerebro.

![Image of sub-menu on barnyard0 index](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/cerebro_delete_barnyard_0.PNG)

Now that it is deleted we can start graylog-server again using `systemctl start graylog-server`. 

## Snort Configuration

Okay, we have Graylog completely configured. The last step is to now pipe logs from Snort into Graylog. Follow the steps below to complete this.

1. Login to pfSense and go to **Services -> Snort**
2. Edit the interface you want to get logs from (most likely your WAN interface)
3. Navigate to **WAN Barnyard2**
4. Check the top box `Enable barnyard2 for this interface. You will also need to enable at least one logging destination below.`
5. Check `Enable logging of alerts to a local or remote syslog receiver.` under `Syslog Output Settings`
6. Set the remote host to your Graylog server IP and set the port to 10001 (Barnyard2 Graylog Input Port)
7. Click `Save` at the bottom

    ![Screenshot of Snort Syslog Config](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/snort_pfsense_logging_configuration.PNG)

## Confirm Logging

We now need to confirm that Graylog is receiving all the logs for Snort. We can do this by going to **Streams -> Snort Barnyard2 Logs** and making sure we're receiving messages. If you click into a message you should see variables such as `src_addr`, `src_addr_geo_location`, `dst_addr`, `dst_addr_geo_location`, etc. 

![Screenshot of Graylog Message Example](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/graylog_snort_message_example.PNG)

## Grafana Configuration

Now that we have logs within Graylog for Snort and we're receiving Geo Location coordinates we can map those coordinates to World Maps within Grafana. To complete this section you'll need:
1. Grafana already running in your environment
2. An understanding of how Grafana Panels work
3. World Map Grafana Plugin

First things first, we need to add Graylog as a source to Grafana. We can do this by adding a new Elasticsearch data source and configuring it like the image below. In the `URL` box put https://X.X.X.X:9200 (replace X.X.X.X with the IP of your Graylog/Elasticsearch server).

![Screenshot of Grafana Datasource Config](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/grafana_elasticsearch_datasource.PNG)

Now that we have our data source we can import the `snort_grafana_dashboard.json` file in this repository to Grafana. This will give you a very basic starting dashboard for Snort that shows an Incoming connection map, top city, top country, top source ip, top classification, top attack and top destination port. 

To import the dashboard:
1. Go to **Dashboards -> Manage**
2. Click **Import** in the top-ish right
3. Click **Upload .JSON** and select the JSON file you downloaded from the repo
4. Click **Load**

You have now uploaded the dashboard but you'll need to edit each panel to target the newly created Elasticsearch data source. Once you've changed the data source for each panel you should be off to the races!

![Screenshot of Grafana Snort Dashboard](https://github.com/shrunbr/graylog_pfsense_barnyard2/blob/master/screenshots/grafana_snort_dashboard.PNG)

Enjoy your new parsed Snort logs and Grafana dashboard!