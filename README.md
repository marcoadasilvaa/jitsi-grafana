#How-to to setup Grafana dashboards to monitor Jitsi, my comprehensive tutorial for the beginner

Docker Grafana: https://dev.to/project42/install-grafana-influxdb-telegraf-using-docker-compose-56e9

Original: https://community.jitsi.org/t/how-to-to-setup-grafana-dashboards-to-monitor-jitsi-my-comprehensive-tutorial-for-the-beginner/38696



Who has read my earlier post about setting up 2 servers with jitsi and jibri (How-to to setup integrated Jitsi and Jibri for dummies, my comprehensive tutorial for the beginner 112), knows about my setup. Now I will show how I was able to setup a working dashboard to monitor the Jitsi Meet server based on the rest api provided by Jitsi. This will require installation of Grafana (dashboards), InfluxDB (performance database) and Telegraf (data collector).

In this example I decided to host my dashboards (all 3 services) on my Jibri-Server since that server is not used very often. In the examples I will connect based on IP-Adresses, you have to carefully substitute these with your own IPv4 adresses:
IPv4 of my Jitsi server: 116.203.231.172
IPv4 of my Grafana server: 116.203.20.99

We will briefly go through following steps for this installation:
Server “grafana” (116.203.20.99)
Step 1: Install database (InfluxDB) to host the data for dashboards
Step 2: Install Grafana to display jitsi stats in dashboards
Step 3: Install service (telegraf) to collect stats and write to database (InfluxDB)

Server: “jitsi” (116.203.231.172)
Step 4: Adapt Jitsi onfiguration to expose stats

At the end, I ended up with this dashboard showing stats during a call with 4 participants:
image
image
1909×946 99.1 KB

This dashboard contains ALL fields from the colibri rest-api response. Will require tweaking and tuning.
Server "grafana"

Step 1: Install InfluxDB

apt update && apt install -y gnupg2 curl wget
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
apt update && apt install influxdb -y
systemctl enable --now influxdb
systemctl status influxdb
If you run a firewall (i.e. ufw) on this server, open the port for influxdb and grafana webserver:

ufw allow 8086/tcp
ufw allow 3000/tcp
Step 2: Install Grafana to display stats dashboards

curl https://packages.grafana.com/gpg.key | sudo apt-key add -
add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
apt update && apt install grafana -y
systemctl enable --now grafana-server
systemctl status grafana-server
Step 3: Install & configure telegraf

wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
echo "deb https://repos.influxdata.com/debian buster stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
apt update && apt install telegraf -y
mv /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.original
nano /etc/telegraf/telegraf.conf

Enter following contents in telegraf.conf:

[global_tags]

###############################################################################
#                                  GLOBAL                                     #
###############################################################################

[agent]
    interval = "10s"
    debug = false
    hostname = "jitsi_host"
    round_interval = true
    flush_interval = "10s"
    flush_jitter = "0s"
    collection_jitter = "0s"
    metric_batch_size = 1000
    metric_buffer_limit = 10000
    quiet = false
    logfile = ""
    omit_hostname = false
nano /etc/telegraf/telegraf.d/jitsi.conf

Enter following contents in jitsi.conf:

###############################################################################
#                                  INPUTS                                     #
###############################################################################

[[inputs.http]]
    name_override = "jitsi_stats"
    urls = [
      "http://116.203.231.172:8080/colibri/stats"
    ]

    data_format = "json"

###############################################################################
#                                  OUTPUTS                                    #
###############################################################################

[[outputs.influxdb]]
    urls = ["http://localhost:8086"]
    database = "jitsi"
    timeout = "0s"
    retention_policy = ""
We enable start on boot and start telegraf now on server “jitsi”:

systemctl enable --now telegraf
systemctl status telegraf
(Mind: We will not create a database as Telegraf will create our database if it does not find one)

Server: "jitsi"

Step 4: Adapt Jitsi onfiguration to expose stats

nano /etc/jitsi/videobridge/config

Make sure to configure the jvb options:

JVB_OPTS="--apis=rest,xmpp"

and

nano /etc/jitsi/videobridge/sip-communicator.properties

Here we configure colibri statistics:

org.jitsi.videobridge.ENABLE_STATISTICS=true
org.jitsi.videobridge.STATISTICS_TRANSPORT=muc,colibri
service jitsi-videobridge2 restart

Check output in the terminal on the jitsi server:

curl -v http://127.0.0.1:8080/colibri/stats

Response: {"inactive_endpoints":0,"inactive_conferences":0,"total_ice_succeeded_relayed":0,"total_loss_degraded_participant_seconds":0,"bit_rate_download":0,"muc_clients_connected":1,"total_participants":0,"total_packets_received":0,"rtt_aggregate":0.0,"packet_rate_upload":0,"p2p_conferences":0,"total_loss_limited_participant_seconds":0,"octo_send_bitrate":0,"total_dominant_speaker_changes":0,"receive_only_endpoints":0,"total_colibri_web_socket_messages_received":0,"octo_receive_bitrate":0,"loss_rate_upload":0.0,"version":"2.1.169-ga28eb88e","total_ice_succeeded":0,"total_colibri_web_socket_messages_sent":0,"total_bytes_sent_octo":0,"total_data_channel_messages_received":0,"loss_rate_download":0.0,"total_conference_seconds":0,"bit_rate_upload":0,"total_conferences_completed":0,"octo_conferences":0,"num_eps_no_msg_transport_after_delay":0,"endpoints_sending_video":0,"packet_rate_download":0,"muc_clients_configured":1,"conference_sizes":[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],"total_packets_sent_octo":0,"conferences_by_video_senders":[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],"videostreams":0,"jitter_aggregate":0.0,"total_ice_succeeded_tcp":0,"octo_endpoints":0,"current_timestamp":"2020-04-17 23:14:38.468","total_packets_dropped_octo":0,"conferences":0,"participants":0,"largest_conference":0,"total_packets_sent":0,"total_data_channel_messages_sent":0,"total_bytes_received_octo":0,"octo_send_packet_rate":0,"conferences_by_audio_senders":[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],"total_conferences_created":0,"total_ice_failed":0,"threads":37,"videochannels":0,"total_packets_received_octo":0,"graceful_shutdown":false,"octo_receive_packet_rate":0,"total_bytes_received":0,"rtp_loss":0.0,"total_loss_controlled_participant_seconds":0,"total_partially_failed_conferences":0,"endpoints_sending_audio":0,"total_bytes_sent":0,"mucs_configured":1,"total_failed_conferences":0,"mucs_joined":1}

With this response we see that the rest api responds and can be used for our purpose!

To make sure that the colibri rest endpoint can be accessed by telegraf:

ufw allow 8080/tcp

Configure dashboards in Grafana

Open Grafana in browser: http://116.203.20.99:3000 29 and prepare your admin account

Add datasource.
We will add an InfluxDB datasource and set it to be the default.

Name: InfluxDB
Default: On
HTTP URL: http://localhost:8086 12
HTTP Access: Server (default)
Database: jitsi
image
image
639×932 41.5 KB
Build a Dashboard.

New Panel: Choose Visualisation
Type: Stat
Go to “Queries”
–> Follow the settings in below screenshot to get a first panel display.
image
image
716×859 50.7 KB
A collection of all fields from the jitsi rest api response is available in the json attached for convenience.

Happy tweaking and tuning…

Cheers, Igor
